# First App in 15 Minutes

> **See also:** [rules.md](rules.md) · [config-json.md](config-json.md) · [js-api.md](js-api.md) · [deploy.md](deploy.md)
> **← [Home](index.md)**

End-to-end: from an empty folder to a working widget in the CRM.
Read [rules.md](rules.md) before starting.

---

## What is a miniapp

An HTML page in an `<iframe>` embedded in the CRM platform. It communicates with the platform via `postMessage` (JS class `VMCRMUserApp`). It can read and write catalog data via `App.fetch()`.

**Only allowed in the zip:** HTML, JS, CSS, images, fonts, md. Server-side code (PHP) is **prohibited**. If you need server logic — use a remote app (`install_url` instead of a zip).

---

## Step 1. Structure

```
my-app/
├── config.json      # required
├── widget.html      # frame (one or more)
├── logo.svg         # optional
```

## Step 2. Minimal `config.json`

```json
{
    "name": "My App",
    "version": "1.0.0",
    "description": "Counts items in a catalog",
    "about": "## What it does\nShows the number of records in a catalog.\n\n## Where it appears in CRM\n- Menu item 'Counter' after the Tasks catalog\n- Widget under the list of any catalog\n\n## How to use\nOpen any catalog — you'll see the counter below the table.",
    "logo": "logo.svg",
    "urls": {
        "widget": "widget.html"
    },
    "urlsConf": {
        "widget": { "method": "get" }
    },
    "permissions": {
        "catalogs": { "*": ["read"] }
    },
    "menu": {
        "tt_tasks": { "frame": "widget", "name": "Counter" }
    },
    "catalogs": {
        "": {
            "catalog.items.footer": {
                "name": "Record Counter",
                "frame": "widget"
            }
        }
    }
}
```

**Important about `menu`:**
- **Key** = catalog alias in the sidebar, **AFTER which** the new item appears. For example `"tt_tasks"` inserts the item right after the Tasks section.
- **Value** — object with `frame` (name from `urls`) and `name` (label in the menu).
- For multiple menu items — specify multiple keys: `"tt_tasks": {...}, "ag_clients": {...}`.

**About `catalogs.""`:**
- Empty key `""` — setting applies to **all catalogs** on the platform. Convenient for universal widgets.
- Specific key (`"tt_tasks"`) — setting only for that catalog.

Full spec for all fields, entry points, and permissions → [config-json.md](config-json.md).

## Step 3. Minimal `widget.html`

```html
<!DOCTYPE html>
<html lang="en">
<head>
<meta charset="UTF-8">
<style>
    body { font-family: 'Open Sans', -apple-system, sans-serif; margin: 0; padding: 12px; overflow: hidden; }
    .count { font-size: 20px; font-weight: 600; color: var(--primary-color, #2e7d32); }
</style>
</head>
<body>
<div id="app">Loading…</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

const params = await App.getRequestParams();
const { catalog } = params.data;

const resp = await App.fetch(`/db/${catalog}.json?page=1`);
const total = resp.total ?? resp.data?.length ?? 0;

document.getElementById('app').innerHTML =
    `<div>Catalog <b>${catalog}</b>: <span class="count">${total}</span> records</div>`;

requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight));
</script>
</body>
</html>
```

Key points:
- `VMCRMUserApp` loads from the portal, not from a CDN — [js-api.md](js-api.md)
- `App.fetch()` uses the user's session, no token required — [data-api.md](data-api.md)
- `App.setFrameSize()` adjusts the iframe height — more on auto-resize and styling in [styling.md](styling.md)

### ⚠️ `/db/` vs `/api/db/` — don't confuse them

Code inside the miniapp (in the iframe) uses **`/db/{catalog}.json`** via `App.fetch`. This works via the **user's session** (cookie), no token needed.

If you want to test the same request from outside (via curl, a script, Postman) — use **`/api/db/{catalog}`** with a Bearer token, not `/db/`:

```bash
# Inside iframe — via App.fetch:
await App.fetch('/db/tt_tasks.json')                       # ← works

# Outside (curl, tests, automation):
curl "https://panel.korfix.ru/api/db/tt_tasks?limit=10" \
  -H "Authorization: Bearer YOUR_TOKEN"                    # ← works

# DON'T do this from outside — you'll get a 302 to login:
curl https://panel.korfix.ru/db/tt_tasks.json              # ✗ 302
```

More details: [data-api.md](data-api.md) — section "Key rule: which endpoint from where".

## Step 4. Package and deploy

```bash
cd my-app
zip -r /tmp/my-app.zip config.json widget.html logo.svg

curl -X POST "https://panel-korfix.vnn.ru/api/db/marketplace/{ID}?token={TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

Getting the app ID, CI/CD script, version updates → [deploy.md](deploy.md).

## Step 5. Install and verify

1. Open the marketplace: `/db/marketplace`
2. Find your app in the list → click **Install**
3. After installation, open any catalog — the counter will appear below the table

> The list of installed apps is available in `/db/installed_apps` (filled automatically), but installation is launched from the marketplace, not from there.

---

## What next

| Need | File |
|------|------|
| Write data to a catalog | [data-api.md](data-api.md) |
| React to record saves | [catalog-rules.md](catalog-rules.md) or [storage-and-hooks.md](storage-and-hooks.md) |
| Build a dashboard widget | [dashboards.md](dashboards.md) |
| Create custom catalogs on install | [self-provisioning.md](self-provisioning.md) |
| See what catalogs are available | [korfix-catalogs.md](korfix-catalogs.md) |
| Style UI to match the platform | [styling.md](styling.md) |
| Before release | [checklist.md](checklist.md) |

---

**Next:** [config-json.md](config-json.md) · **← [Home](index.md)**
