# Deploying a Miniapp

> **See also:** [rules.md](rules.md) · [getting-started.md](getting-started.md) · [checklist.md](checklist.md) · [config-json.md](config-json.md)
> **← [Home](index.md)**

Step-by-step: creating, packaging, uploading via API, and updating miniapps.

> ⚠ **Important:** deploying and updating a miniapp always goes through the `/db/marketplace` catalog (create/re-upload a zip). The `/db/installed_apps` catalog is a **read-only registry of installed** apps, filled automatically. Don't write to it manually or via API.

---

## Step-by-step app creation

### Step 1: Determine the type

| Type | When to use |
|------|-------------|
| Footer widget in catalog | Additional visualization below the list |
| Card widget | Extra info on the item view page |
| Menu item | Full-screen app (calendar, kanban) |
| itemsAction | Action on an item (popup) |
| afterSave hook | Server-side reaction to saves (remote only) |

### Step 2: Create files

```bash
mkdir my-app
cd my-app
# Create config.json and widget.html
```

### Step 3: Write config.json

Determine:

- `urls` — app frames
- `urlsConf` — request method (`get` for local, `post` for remote)
- `catalogs` or `menu` — where it embeds
- `permissions` — which catalogs it reads/writes

Full spec and example → [config-json.md](config-json.md).

### Step 4: Write the HTML frame

Template:

```html
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Open Sans', -apple-system, sans-serif; background: transparent; }
    </style>
</head>
<body>
<div id="app">Loading...</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

async function init() {
    const params = await App.getRequestParams();
    const { catalog, itemId } = params.data;

    // Load data
    const resp = await App.fetch(`/db/${catalog}.json`);

    // Render UI
    document.getElementById('app').innerHTML = `
        <p>Catalog: ${catalog}, items: ${resp.data?.length || 0}</p>
    `;

    // Adjust frame height
    App.setFrameSize(null, document.body.scrollHeight + 10);
}

init();
</script>
</body>
</html>
```

### Step 5: Check against the checklist

Before packaging, go through [checklist.md](checklist.md). Pay special attention to:

- `config.json` is valid, `about` field has all 5 sections
- All files from `urls` exist
- `App.setFrameSize()` is called after rendering
- `App.fetch()` instead of `window.fetch()`
- `font-size ≥ 16px` in input/select/textarea (iOS Safari)

### Step 6: Package into zip

> ✅ **The manifest is validated server-side on deploy.** Both `POST /api/db/marketplace/{id}`
> and `POST /api/marketplace/deploy/{id}` validate the bundled `config.json` and the archive,
> and return the verdict in the response — read it and fix, no need to open the app to discover
> a broken manifest.
>
> **Errors block the deploy** (response `{"status":"error","message":"...list..."}`). Triggered by:
> invalid JSON; missing `name`; missing or non-object `urls`; any file referenced in `urls.*` or
> `logo` that isn't present in the zip. The `message` lists every problem at once. Example:
> ```json
> {"status":"error","message":"config.json: invalid JSON; field \"name\" is required; urls.widget -> widget.html not found in zip"}
> ```
>
> **Warnings don't block** — the deploy succeeds and the response carries `warnings`. Triggered by
> missing recommended fields: `package`, `permissions`, `about`. Example:
> ```json
> {"status":"success","id":"123","alias":"abc...","warnings":["field \"package\" is recommended","field \"about\" is recommended"]}
> ```
>
> Still worth a local pre-flight before zipping:
> ```bash
> python3 -m json.tool config.json   # or: jq . config.json
> ```

```bash
# From the app directory:
zip -r /tmp/my-app.zip config.json widget.html *.js *.css *.svg
```

**Allowed extensions in zip**: html, htm, txt, js, png, jpg, jpeg, webp, ico, gif, css, json, svg, map, md, eot, woff2, ttf, woff.

**Prohibited**: php, exe, sh — the platform will reject the zip.

### Step 7: Upload to the marketplace

Uploading an app = creating/updating a record in the `/db/marketplace` catalog.

#### Via the UI

1. Open `/db/marketplace`
2. Click **Add** (for a new app) or open an existing one (to update)
3. Upload the zip file in the `doc1` field
4. Save

#### Via API (recommended for vibe-coding)

**Creating a new app** (no ID yet):

```bash
curl -s -X POST "https://panel.korfix.ru/api/db/marketplace/add" \
  -H "Authorization: Bearer {TOKEN}" \
  -F "name=My App Name" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
# Response: {"status":"success","id":"123","alias":"abc..."}
# Save the ID — you'll need it for subsequent updates
```

Fields are passed **without the `form[]` wrapper** — just `name=...`. The name, description, and tags will also be pulled from `config.json` inside the archive on the next render.

**Updating an existing app** (ID known):

```bash
curl -s -X POST "https://panel.korfix.ru/api/db/marketplace/{ID}" \
  -H "Authorization: Bearer {TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

The standard endpoint uploads the zip and notifies the store via an internal hook. In production configuration this is sufficient.

**Shorthand via `deploy`** (update + refresh in one request):

```bash
# /api/marketplace/deploy/{ID} — update + refresh in one call.
# Useful when you need to explicitly clear appconfig (e.g. in local configuration).
curl -s -X POST "https://panel.korfix.ru/api/marketplace/deploy/{ID}" \
  -H "Authorization: Bearer {TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

> Limitation: `deploy` requires an existing ID. For creating a new app — use standard `POST /api/db/marketplace/add`.

**Cache-only invalidation** (when the zip is already uploaded):

```bash
curl -s -X POST "https://panel.korfix.ru/api/marketplace/refresh/{ID}" \
  -H "Authorization: Bearer {TOKEN}"
```

> `token=` in the query string also works as an alternative to the Bearer header.

### Step 8: Install the app

Installation = a user action in the marketplace UI.

1. Open the marketplace: `/db/marketplace`
2. Find your app in the list (by `name` from `config.json`)
3. Click the **Install** button on the app card
4. Frames will appear at the entry points specified in `config.json`

> The `installed_apps` registry is filled automatically — don't write to it manually.

---

## Auto-deploy via API

Zip uploads can be automated via the REST API without touching the UI.

### Full cycle: create + deploy in one command

```bash
zip -r /tmp/my-app.zip config.json widget.html *.js *.css

# First time — create and get the ID:
curl -s -X POST "https://panel.korfix.ru/api/db/marketplace/add" \
  -H "Authorization: Bearer {TOKEN}" \
  -F "name=My App" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
# → {"status":"success","id":"123","alias":"abc..."}

# Subsequent updates by ID:
curl -s -X POST "https://panel.korfix.ru/api/db/marketplace/123" \
  -H "Authorization: Bearer {TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

### Getting an existing app's ID

```bash
curl -H "Authorization: Bearer {TOKEN}" \
  "https://panel.korfix.ru/api/db/marketplace?filter[name]=My App"
```

### Verifying the version after deploy

```bash
curl -H "Authorization: Bearer {TOKEN}" \
  "https://panel.korfix.ru/api/db/marketplace/{ID}"
# The appconfig.version field contains the version from config.json
```

### CI/CD script

```bash
#!/bin/bash
APP_DIR="./my-app"
API_URL="https://panel.korfix.ru"
TOKEN="your-api-token"
APP_ID="50"

cd "$APP_DIR"
zip -r /tmp/app-deploy.zip config.json *.html *.js *.css 2>/dev/null

RESPONSE=$(curl -s -X POST "$API_URL/api/db/marketplace/$APP_ID" \
  -H "Authorization: Bearer $TOKEN" \
  -F "doc1=@/tmp/app-deploy.zip;type=application/zip")

echo "$RESPONSE"
```

### Token requirements

The token from `/db/api` must have access to the `marketplace` catalog (POST method). Add `db_marketplace_post` to the token's **"API Classes"**.

### Vibe-coding cycle

1. Assistant edits the app's html/js/css files
2. Checks against [checklist.md](checklist.md)
3. Packages into zip
4. Deploys via `curl` in one command to `/db/marketplace`
5. Result is visible in the browser immediately after page refresh

---

## Updating an app

Update process:

1. Assistant edits the app's html/js/css files
2. Checks against [checklist.md](checklist.md)
3. Packages into zip
4. Re-uploads zip (via UI or API deploy) to `/db/marketplace`
5. Marketplace shows an "update" badge
6. User clicks the **Update** button on the app page
7. App frames start serving the new code

> With API deploy, steps 1-4 are automatic. Steps 5-7 are user actions in the marketplace UI.

---

**Next:** [checklist.md](checklist.md) · **← [Home](index.md)**
