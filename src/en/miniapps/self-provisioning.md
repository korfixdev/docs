# Self-provisioning: Creating Data Structures

> **See also:** [data-api.md](data-api.md) · [catalog-rules.md](catalog-rules.md) · [catalog-settings.md](catalog-settings.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

An app can create its own catalogs (tables) and custom fields on install or first run.

---

## Token / permission requirements

For self-provisioning from the miniapp (`App.fetch` via session) — no extras needed, the user session covers it.

For Bearer token access (scripts, CI, external agents) the token needs these classes:

| Class | Purpose |
|---|---|
| `db_custom_dbtables_get` | Check catalog existence in the registry |
| `db_custom_dbtables_post` | Create a custom catalog |
| `db_custom_dbfields_post` | Create catalog fields |
| `db_access_db_get` | Read the auto-created permissions record (for subsequent update) |
| `db_access_db_post` | Update permissions for roles (`configureAccess` helper) |

If any class is missing — self-provisioning will fail with 403 on the first request. Developer agents should run [`korfix-token-audit`](../../devkit-skills/korfix-token-audit/) before starting to avoid hitting this mid-process.

---

### Built-in installer (recommended)

The app checks for the catalog on first run and prompts an install button.
No tokens or terminal access needed — works via session auth (`App.fetch()`).

#### Checking catalog existence

**Important**: don't check custom catalog existence via `/db/{catalog}.json` —
when the catalog doesn't exist, CRM doesn't return an error; instead it falls back to a default catalog
(returns data from another catalog with `status: "ok"`). The app would incorrectly decide the catalog exists and skip the install screen.

**Correct way** — check via the `custom_dbtables` registry:

```js
const CATALOG = 'custom_quicknotes';

async function checkCatalogExists() {
    try {
        // dbname = table name without custom_ prefix (quicknotes, not custom_quicknotes)
        const resp = await App.fetch('/db/custom_dbtables.json?form[dbname]=quicknotes');
        return !!(resp && resp.data && Array.isArray(resp.data) && resp.data.length > 0);
    } catch (e) {}
    return false;
}
```

For apps with multiple catalogs — a universal function:

```js
async function checkCatalogExists(catalogName) {
    try {
        const tablename = catalogName.replace('custom_', '');
        const resp = await App.fetch('/db/custom_dbtables.json?form[dbname]=' + tablename);
        return !!(resp && resp.data && Array.isArray(resp.data) && resp.data.length > 0);
    } catch (e) {}
    return false;
}
```

#### Creating a catalog and fields

> **Required field `scheme`** when creating `custom_dbtables`.
> This is a template schema — a set of base fields (`id`, `alias`, `name`, `ts`, `hidden`, `from_auth`, `from_group`) on which the physical database table is built. Without this field the platform doesn't know how to build the table and rejects the request.
>
> Currently one value is available: `coredb_def_catalog` — the default catalog schema. The platform may add more options in future; current list — via `App.fetch('/db/custom_dbtables/sheme.json')` → `scheme.arr` field.
>
> **Don't confuse** with the `scheme` field in `custom_dbfields` — there it's a reference to the target catalog (`custom_X`) for which the field is being created.

```js
const FIELDS = [
    { name: 'Note text',  dbname: 'content',   type: 'textarea', f_maxlen: 65535 },
    { name: 'Priority',   dbname: 'priority',   type: 'select',   f_arr: 'normal\nimportant\nurgent', f_default: 'normal' },
    { name: 'Status',     dbname: 'status',     type: 'select',   f_arr: 'active\ndone\narchive', f_default: 'active' },
    { name: 'Deadline',   dbname: 'due_date',   type: 'datetime' },
];

// Get current user ID (for from_auth/from_group)
let currentUserId = 0;
async function loadUserId() {
    const schema = await App.fetch('/db/custom_dbtables/sheme.json');
    const arr = schema?.data?.from_auth?.arr || {};
    currentUserId = Object.keys(arr).find(k => k !== '0') || 0;
}

function uid() {
    return Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
}

async function createTable() {
    return App.fetch('/db/custom_dbtables/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[alias]': uid(),
            'form[name]': 'Quick Notes',
            'form[dbname]': 'quicknotes',
            'form[scheme]': 'coredb_def_catalog',  // REQUIRED: catalog template schema
            'form[from_auth]': currentUserId,
            'form[from_group]': currentUserId,
            submit: 1
        }
    });
}

async function createField(field) {
    const body = {
        'form[alias]': uid(),
        'form[name]': field.name,
        'form[dbname]': field.dbname,
        'form[type]': field.type,
        'form[scheme]': CATALOG,
        'form[from_auth]': currentUserId,
        'form[from_group]': currentUserId,
        submit: 1
    };
    if (field.f_maxlen) body['form[f_maxlen]'] = field.f_maxlen;
    if (field.f_arr)     body['form[f_arr]'] = field.f_arr;
    if (field.f_default) body['form[f_default]'] = field.f_default;

    return App.fetch('/db/custom_dbfields/add?edit&ajax=1', { method: 'POST', body });
}

async function runInstall() {
    await createTable();
    for (const field of FIELDS) {
        await createField(field);
    }

    // REQUIRED: configure permissions in access_db, otherwise the catalog is invisible to regular roles.
    // Default = 2 (self — each user sees only their own records) — typical best-default.
    // For collaborative catalogs (shared tasks, clients) — pass 1.
    // configureAccess function is defined in "Access permissions (access_db)" section below.
    await configureAccess(CATALOG);  // CATALOG = 'custom_quicknotes'
}
```

> Without `configureAccess` after `createTable`, the catalog is visible only to admins — regular roles get empty `data: []` (see the "Access permissions (access_db)" section below).

#### Full pattern: HTML install screen (ready-made UX)

A correct install screen should:

- Save an **install log** to `App.storage` — so on re-open the user can see what was done
- On re-open (catalog already exists) — offer **"Reinstall"** or **"Close"** — not just show a blank screen
- Handle errors and show a "Retry" button

```html
<!-- Install screen (hidden by default) -->
<div id="installScreen" style="display:none;">
    <h2 id="installTitle">App Installation</h2>
    <p id="installIntro">A data catalog is needed. Click the install button.</p>
    <div class="install-actions">
        <button id="btnInstall" class="btn btn-primary">Install data structure</button>
        <button id="btnReinstall" class="btn btn-secondary" style="display:none;">Reinstall</button>
        <button id="btnClose" class="btn btn-stroke" style="display:none;">Close</button>
    </div>
    <pre id="installLog" style="background:#f5f5f5;padding:12px;margin-top:16px;max-height:300px;overflow:auto;font-size:12px;white-space:pre-wrap;"></pre>
</div>

<!-- Main UI (hidden by default) -->
<div id="mainUI" style="display:none;">
    <!-- ... main app interface ... -->
</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

const LOG_KEY = 'install.log';  // App.storage key for saving the log

// ---- helpers ----

function logLine(text) {
    const el = document.getElementById('installLog');
    const line = `[${new Date().toISOString().slice(11,19)}] ${text}\n`;
    el.textContent += line;
    el.scrollTop = el.scrollHeight;
}

async function saveLog() {
    const log = document.getElementById('installLog').textContent;
    await App.storage.set(LOG_KEY, log);
}

async function loadSavedLog() {
    const rec = await App.storage.get(LOG_KEY);
    return rec?.value ?? '';  // ← correct read via .value
}

// ---- init ----

async function init() {
    const exists = await checkCatalogExists();

    if (!exists) {
        // First run — clean install screen, only Install button
        document.getElementById('installScreen').style.display = '';
        return;
    }

    // Catalog exists → show mainUI, but give access to "reinstall"
    document.getElementById('mainUI').style.display = '';
    loadData();

    // "⚙ Reinstall" button somewhere in settings → opens installScreen with log
    // Add handler for the gear icon here
}

async function openInstallScreen(isReinstall = false) {
    document.getElementById('mainUI').style.display = 'none';
    document.getElementById('installScreen').style.display = '';

    // Show saved log from previous install
    const savedLog = await loadSavedLog();
    document.getElementById('installLog').textContent = savedLog;

    if (isReinstall) {
        document.getElementById('installTitle').textContent = 'App Reinstallation';
        document.getElementById('installIntro').textContent = 'Structure already created. You can reinstall (recreates fields) or close.';
        document.getElementById('btnInstall').style.display = 'none';
        document.getElementById('btnReinstall').style.display = '';
        document.getElementById('btnClose').style.display = '';
    }
}

// ---- button handlers ----

document.getElementById('btnInstall').addEventListener('click', async () => {
    const btn = document.getElementById('btnInstall');
    btn.disabled = true;
    btn.textContent = 'Installing...';
    logLine('Starting installation...');

    try {
        await runInstall((msg) => logLine(msg));  // runInstall calls logLine for each step
        logLine('✓ Installation complete');
        await saveLog();

        // Go to main UI
        document.getElementById('installScreen').style.display = 'none';
        document.getElementById('mainUI').style.display = '';
        loadData();
    } catch (e) {
        logLine(`✗ Error: ${e.message}`);
        await saveLog();
        btn.disabled = false;
        btn.textContent = 'Retry installation';
    }
});

document.getElementById('btnReinstall').addEventListener('click', async () => {
    if (!confirm('Reinstall? This will recreate missing fields. Data in existing records will be preserved.')) return;
    const btn = document.getElementById('btnReinstall');
    btn.disabled = true;
    btn.textContent = 'Reinstalling...';
    logLine('--- Reinstallation ---');

    try {
        await runInstall((msg) => logLine(msg));
        logLine('✓ Reinstallation complete');
        await saveLog();
        alert('Done');
        document.getElementById('installScreen').style.display = 'none';
        document.getElementById('mainUI').style.display = '';
    } catch (e) {
        logLine(`✗ Error: ${e.message}`);
        await saveLog();
        btn.disabled = false;
        btn.textContent = 'Retry reinstallation';
    }
});

document.getElementById('btnClose').addEventListener('click', () => {
    document.getElementById('installScreen').style.display = 'none';
    document.getElementById('mainUI').style.display = '';
});

init();
</script>
```

**Key points:**

1. `LOG_KEY = 'install.log'` — log saved in `App.storage` (isolated app storage), survives browser restart
2. On re-open of installScreen — saved log is loaded via `loadSavedLog()` (read via `.value`, not directly)
3. Two buttons: **"Install"** for first run and **"Reinstall"** for re-run with existing catalog
4. **"Close"** button — to exit installScreen without action
5. `runInstall(cb)` takes a callback for logging each step (`logLine('Creating field X...')` from inside)
6. Log is saved even on error — can be reviewed later

### External script (curl / CI/CD)

Creating structure via terminal with Bearer token:

```bash
# Create catalog
curl -s -X POST "$API_URL/api/db/custom_dbtables" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"form": {"name": "Quick Notes", "dbname": "quicknotes"}}'

# Add field
curl -s -X POST "$API_URL/api/db/custom_dbfields" \
  -H "Authorization: Bearer $TOKEN" \
  -d 'form[name]=Note text&form[dbname]=content&form[type]=textarea&form[scheme]=custom_quicknotes&submit=1'
```

### Field types

| `type` | Description |
|--------|-------------|
| `textbox` | Single-line text |
| `textarea` | Multi-line text |
| `select` | Dropdown list |
| `checkbox` | Checkbox |
| `datetime` | Date and time |
| `photo` | File upload |
| `select_from_table` | Select from another catalog |

Values for `select`: pass in `f_arr` separated by `\n` (newline).

**Important**: in JS code, custom catalog fields are accessed with the `custom_` prefix — e.g. `note.custom_content`, `note.custom_priority`.

---

## Access permissions (access_db) — REQUIRED

### What access_db is and what it is NOT

**`access_db` is not authentication.** The user is logged into Korfix, they have a session or Bearer token, `App.fetch('/db/...')` works — this is **authentication**.

**`access_db` is row-level visibility of the catalog per role.** Even if the user is logged in and can physically make a request — the platform will filter records (or return empty data) based on their `account_type` and the `access_db` settings for that catalog.

| Mechanism | Controls | Configured in |
|---|---|---|
| **Session / Bearer** | Whether the request can be executed at all (401 if not) | User login or `/db/api` token |
| **`permissions` in config.json** | What the miniapp can do via `App.fetch` (sandbox) | `config.json` → `permissions.catalogs` |
| **`access_db`** | **Which records a user with a given role can see** in the catalog (0/1/2 — none/all/own) | Catalog `/db/access_db`, one record per (catalog × tenant) |

Without a correct `access_db` record, the miniapp **will get an empty list** (or HTTP 200 with `data: []`), even if the API request formally succeeded and the user has a session. This is a common cause of "I followed the docs but the list is empty".

### Auto-creation result

When a record is created in `custom_dbtables` (via UI, API/Bearer, or `App.fetch` from the miniapp) — an afteradd hook fires that **automatically creates a record in `access_db`** with permissions **for administrators only**: `acctype_root=1, acctype_adm=1`, all other roles = `0` (no access).

This means:

- The catalog is **immediately visible** to users with root/adm role (usually the app creator — an admin)
- **Not visible** to other roles (manager, operator, client, etc.) — they get empty `data: []` from `App.fetch('/db/custom_X.json')`
- If the miniapp is intended for other roles — the app **must update** `access_db` with the appropriate `acctype_*` values (see `configureAccess` pattern below)

One physical table `crm__custom_{dbname}` is shared between accounts (different `from_group`), but `access_db` is created per-account — so each account independently configures permissions for its roles.

`access_db` has a unique constraint `UNIQUE (dbmodule, from_auth, from_group)`. For one catalog in one account — **one record** (with `from_auth = catalog owner ID`, `from_group = account ID`). Permissions are set via `acctype_*` columns — don't create additional records to "expand access".

!!! warning "Anti-pattern: `from_auth = 0` in access_db"

    In regular catalogs `from_auth = 0` means "record belongs to the whole group" (visible to all members of `from_group`), and that's a legitimate pattern.

    **In `access_db` this must not be done.** Catalog visibility per role is determined by `acctype_*` values (`0`/`1`/`2`) — the row-ownership mechanism via `from_auth` is **not applied by the platform** for this table. Creating a second record with `(dbmodule, 0, from_group)` "to make it visible to all" is pointless — it only adds a duplicate in `/db/access_db`, without granting any real permissions.

    Correct: one record `(dbmodule, owner_id, from_group)` with appropriate `acctype_*`.

**Important — server-side from_auth/from_group substitution:**

Since April 2026 the platform (when `FEATURES_USED.auth_role` is enabled) automatically substitutes `from_group` and `from_auth` from the session/token on INSERT/UPDATE to any catalog. Clients no longer **need** to pass them manually in `form[from_auth]`/`form[from_group]` — the server takes values from auth. If the client does pass them, these values go through validation:

- Non-admins cannot specify another's `from_group` / another's `from_auth` — server replaces with their own
- Admins can explicitly assign an owner (`from_auth = specific ID`) or transfer to another group
- `from_auth = 0` is always allowed (group-wide) — for `access_db` this is an anti-pattern (see above)

This eliminates the old pain of "records created with `from_group=0` if not passed manually". Manual passing is now only for explicit transfers (admin).

### When the app must think about access_db

When designing a miniapp **always decide** which roles should see the catalog and how:

- **Admin only** — leave as-is (default). Suitable for service/config catalogs.
- **Personal data for all roles** — each role → `acctype_* = 2` (self, only own records). Suitable for notes, ToDo, settings, drafts.
- **Collaboration** — selected roles → `acctype_* = 1` (all org records). Suitable for tasks, clients, deals.
- **Mixed** — per-role (admin sees all, client only own, analyst read-only — via additional permission mechanisms).

If from the task context **it's not clear** which scheme to choose — **ask the user**:
> "Which role is this catalog for? Visible only to admin, or should different roles see their own records, or should everyone see everything?"

Don't guess. The correct `access_db` configuration depends on this.

### Updating permissions via API

The `configureAccess` pattern — updates (or creates if missing) the `access_db` record for all roles of the current instance with one value:

```js
async function configureAccess(catalog, defaultValue = 2) {
    const schema = await App.fetch('/db/access_db/sheme.json');
    const acctypeFields = Object.keys(schema.data || {})
        .filter(k => k.startsWith('acctype_'));

    // Check if record exists
    const existing = (await App.fetch(
        `/db/access_db.json?form[dbmodule]=${catalog}`
    )).data?.[0];

    const body = {
        'form[dbmodule]': catalog,
        'form[name]': catalog,
        submit: 1,
    };
    for (const field of acctypeFields) {
        body[`form[${field}]`] = defaultValue;
    }

    let resp;
    if (!existing) {
        // Create new record
        body['form[alias]'] = Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
        resp = await App.fetch('/db/access_db/add?edit&ajax=1', {
            method: 'POST',
            body
        });
    } else {
        // Update existing
        body['form[id]'] = existing.id;
        body['form[alias]'] = existing.alias;
        resp = await App.fetch(`/db/access_db/${existing.alias}?edit&ajax=1`, {
            method: 'POST',
            body
        });
    }

    if (!resp || resp.status === 'error' || resp.status === 'no') {
        throw new Error(
            `configureAccess(${catalog}) failed: ${resp?.message || JSON.stringify(resp)}`
        );
    }
}

// In installer:
await configureAccess('custom_quicknotes');           // all roles → self (2)
await configureAccess('custom_shared_tasks', 1);      // all roles → all (1)
```

The function automatically fetches the current instance's role list — don't hardcode `acctype_adm`, `acctype_b2b2`, etc.

### access_db structure

| Field | Value |
|-------|-------|
| `dbmodule` | Catalog alias **with** `custom_` prefix (`custom_my_catalog`) |
| `name` | Name (usually same as `dbmodule`) |
| `from_group` | User's tenant (from `getUser().group`) |
| `acctype_root` | Permissions for "Administrator" role (0/1/2) |
| `acctype_adm` | Permissions for "Manager" role (0/1/2) |
| `acctype_res`, `acctype_fin`, `acctype_ag1`..`ag6`, `acctype_ec1`..`ec5`, `acctype_b2b1`..`b2b3`, `acctype_md1`..`md3` | Permissions for corresponding roles |

**`acctype_*` values:**

| Value | Access | When to use |
|-------|--------|-------------|
| `0` | No access (catalog hidden) | Role must not see this catalog at all |
| `1` | All org records (own `from_group`) | Collaboration: tasks, clients, deals — visible to all staff |
| `2` | Only own (`from_auth = own user_id`) | Personal data: notes, drafts, settings |

The role list is **instance-specific** — `panel.korfix.ru` has one set, self-hosted may have another. Get the current list: `App.fetch('/db/access_db/sheme.json')` → fields with `acctype_` prefix.

### Best practice: default "2 for all roles" (self-access)

**The safest and most common default — give all roles value `2` (only own records).** This is sufficient for most apps: each user sees only what they created.

When to choose otherwise:
- **Value `1`** (all org records) — for collaborative catalogs: tasks (all see all company tasks), CRM clients, deals, warehouse
- **Value `0`** (no access) — for technical/internal catalogs that a specific account type shouldn't see
- **Mixed** — if roles genuinely differ (admin sees all, manager sees all in their group, client sees only their requests)

---

## MCP access to custom catalogs after install

When an MCP agent works via Bearer token — it sees only the catalogs from the token's `custom_catalogs` field. Catalogs created via self-provisioning **are not added there automatically**.

### Method 1 — Manual (admin UI)

After install: `/db/api` → find the token → **"Access to custom catalogs"** field → select the catalog → save.

### Method 2 — Automatically from the install frame

Requires: token has `db_api_get` + `db_api_post` in `apiclasses_id`.

```js
async function registerCatalogForMCP(catalogAlias, tokenAlias) {
    if (!tokenAlias) return
    const apiResp = await App.fetch(`/db/api.json?form[alias]=${encodeURIComponent(tokenAlias)}`)
    const apiRecord = apiResp?.data?.[0]
    if (!apiRecord) return

    const existing = (apiRecord.custom_catalogs || '').split(',').map(s => s.trim()).filter(Boolean)
    const toAdd = [`db_${catalogAlias}_get`, `db_${catalogAlias}_post`].filter(a => !existing.includes(a))
    if (!toAdd.length) return

    const resp = await App.fetch(`/db/api/${apiRecord.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[id]': apiRecord.id,
            'form[alias]': apiRecord.alias,
            'form[custom_catalogs]': [...existing, ...toAdd].join(','),
            submit: 1
        }
    })
    if (!resp || resp.status === 'error' || resp.status === 'no') {
        throw new Error(`registerCatalogForMCP failed: ${resp?.message || JSON.stringify(resp)}`)
    }
}

// In runInstall() — optional step, wrap in try/catch:
try {
    await registerCatalogForMCP('custom_quicknotes', tokenAlias)
    logLine('✓ Catalog registered for MCP')
} catch (e) {
    logLine(`⚠ MCP registration skipped: ${e.message} — add manually in /db/api`)
}
```

`tokenAlias` — alias of the token to update. Don't hardcode in code: ask the user in the install form or pass via frame parameters.

---

**Next:** [styling.md](styling.md) · **← [Home](index.md)**
