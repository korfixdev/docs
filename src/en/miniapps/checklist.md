# Pre-release Checklist

> **See also:** [deploy.md](deploy.md) · [rules.md](rules.md) · [styling.md](styling.md) · [self-provisioning.md](self-provisioning.md)
> **← [Home](index.md)**

Check all items before publishing the app to the marketplace.

---

## config.json

- [ ] Valid JSON — the manifest is validated server-side on deploy (config.json + archive). Fix whatever the deploy response reports: `errors` block the deploy (`{"status":"error","message":"..."}`), `warnings` (`"warnings":[...]`) succeed but should still be addressed. Pre-flight locally before zipping: `python3 -m json.tool config.json` / `jq . config.json`
- [ ] `name` field — app name
- [ ] `package` field — package name (app folder name) — recommended; missing → deploy warning
- [ ] `category` field — integer id `1..5` (`1` AI-agents, `2` Business, `3` Games, `4` Tools, `5` Other). The platform writes it to the DB on first install if still empty
- [ ] `description` field — short description (1-2 sentences)
- [ ] `about` field — detailed description with all sections (What it does, Where it appears, Features, How to use, Setup)
- [ ] In `about` — direct links to CRM sections where the app appears (e.g. `/db/tt_tasks`, `/db/ag_cashflows`)
- [ ] `logo` field — path to SVG/PNG icon in the zip
- [ ] All URLs in `urls` — relative (for local) or absolute (for remote)
- [ ] `method: "get"` specified in `urlsConf` for local frames
- [ ] If the app should open from the menu or have an Install button — **`urls.main` declared**. Without it the app won't open from the menu and has no installer entry point
- [ ] `permissions` section declared — all used catalogs and operations listed

## Frames

- [ ] **`install` frame** (if self-provisioning):
  - [ ] Every mutating `App.fetch` → response status checked: `if (!resp || resp.status === 'error' || resp.status === 'no') throw new Error(...)`
  - [ ] Log saved to `App.storage('install.log')` after each step
  - [ ] On re-open shows saved log + "Reinstall" / "Close" buttons
  - [ ] If the app has `urls.widget` — at the end of install, auto-install widget on the first dashboard
  - [ ] After successful install — navigate to `main`: `App.navigate('/db/installed_apps/${token}?frame=main')`
- [ ] **`main` frame** (if `urls.install` exists):
  - [ ] On load, checks catalog existence via `custom_dbtables`
  - [ ] If not installed — `App.navigate('/db/installed_apps/${token}?frame=install')`
- [ ] **`widget` frame** (if `urls.widget` exists):
  - [ ] `permissions.catalogs` contains `"dashboard_widgets": ["read", "write"]`
  - [ ] Default width ≤ 6, compact layout

Full pattern → [frames.md](frames.md)

## Responsiveness and mobile

- [ ] Responsive layout — tables on mobile display as tiles/cards
- [ ] Font size in form fields (input, select, textarea) — **at least 16px** (otherwise iOS Safari zooms on focus)
- [ ] Clickable elements (tabs, buttons) — `<a>` or `<button>`, not `<div>` (iOS doesn't fire click on div)
- [ ] Tested on a mobile device

## UI and UX

- [ ] Gear icon (⚙) in the interface — opens app settings or the install/self-provisioning screen with log
- [ ] Self-provisioning: install screen with progress, result saved in `App.storage` (flag `installed: true`)
- [ ] Input forms: minimal strictness for required fields — don't require what isn't necessary
- [ ] App works with empty data (empty catalog) — shows a meaningful state, not an error

## Code

- [ ] No prohibited files in the zip (php, exe, sh, etc.)
- [ ] `config.json` is in the archive root
- [ ] `App.setFrameSize()` called after rendering to adjust height
- [ ] Auto-resize: `body { overflow: hidden }` + `requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight))`
- [ ] External APIs called via `App.fetch()`, not directly
- [ ] Platform resources (avatars, catalog files, app icons) use **absolute paths** (`/reimg/data/auth/...`, `/data/db/f_{catalog}/...`) — relative paths in the iframe resolve to the app's store-URL archive, not the CRM domain
- [ ] `/db/` requests use `form[]`, `/api/db/` — without `form[]`
- [ ] Styles don't depend on platform CSS (iframe is isolated)
- [ ] Self-provisioning checks catalog via `custom_dbtables`, not `/db/{catalog}.json`
- [ ] If custom catalogs are created — `registerCatalogForMCP` is present, or the install UI has instructions on adding catalog access to the token (otherwise the catalog won't be visible via MCP)
- [ ] **`form[scheme]='coredb_def_catalog'` is passed when creating a catalog** — required field, base catalog template schema. Without it the system won't create the table. Current list — via `/db/custom_dbtables/sheme.json`
- [ ] **`custom_` prefix everywhere** when accessing own catalogs and fields:
    - URL paths: `App.fetch('/db/custom_my_catalog.json')`, not `/db/my_catalog.json`
    - Reading record fields: `record.custom_my_field`, not `record.my_field`
    - In `permissions.catalogs`: `"custom_my_catalog": ["read", "write"]`
    - In `config.json` entry points (`menu`, `catalogs.*`): also with `custom_`
    - **Exception** — when creating the catalog itself in `custom_dbtables.dbname` the prefix is **not specified** (the platform adds it), but when creating a field in `custom_dbfields.scheme` — it **is specified**
- [ ] **`access_db` permissions — access scheme consciously chosen**:
    - The platform **automatically creates** an access_db record (root+adm=1, others=0) right after INSERT into custom_dbtables — **update the existing record**, don't create a new one
    - Table has `UNIQUE (dbmodule, from_auth, from_group)` — one record per `(catalog, owner, group)`. Role visibility is set via `acctype_*` columns, not duplicate records
    - **Anti-pattern:** `from_auth=0` in `access_db` (for other catalogs this means "shared for group", but in `access_db` access is encoded in `acctype_*` — such a record does nothing)
    - Before development: decide / ask user: for which role is the catalog? personal data? collaboration?
    - Personal (each sees only their own) → `configureAccess(catalog, 2)`
    - Collaborative (all see all) → `configureAccess(catalog, 1)`
    - Admin-only → keep default, explicitly mention in `about`
    - The `configureAccess` helper in the `korfix-self-provisioning` skill — fetches the actual acctype_* list from the instance schema (don't hardcode)
- [ ] For bulk record creation — explicitly generate `form[alias]`. `form[from_auth]`/`form[from_group]` are **not required** to pass — the server (with `FEATURES_USED.auth_role`) fills them from session/token automatically; only pass when an admin explicitly transfers a record to another group or assigns an owner

## After installation

- [ ] If the app has a full-screen frame (menu item) — documentation includes a direct link to the page
- [ ] If the app is a dashboard widget (`app-frame`) — an example widget created on the "Main" workspace
- [ ] If the app embeds into a catalog — `about` lists all entry points with links

---

**← [Home](index.md)**
