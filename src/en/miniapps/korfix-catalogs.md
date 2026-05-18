# Marketplace — Korfix Catalogs

> **See also:** [data-api.md](data-api.md) · [self-provisioning.md](self-provisioning.md) · [db-views.md](db-views.md) · [catalog-settings.md](catalog-settings.md)
> **← [Home](index.md)**

Korfix is a full-featured ERP platform. All catalogs are accessible from an app
via `App.fetch('/db/{catalog}.json', ...)` with the current user's permissions.

---

## ⚠️ Catalog Names — Always Module-Prefixed

All Korfix catalogs are named with a module prefix: `ag_*`, `b2b_*`, `md_*`, `crm_*`, `wh_*`, `tt_*`, etc.

**Generic names without prefix do not exist:**

- ❌ `/db/clients` — no such catalog. Likely means `crm_clients` (general CRM), `ag_clients` (AG finance counterparties), `b2b_clients` (B2B clients), or `md_clients` (MD clients). **Ask the user which one** — don't guess.
- ❌ `/db/users` — no. Available: `auth_pers` (all accounts), `ag_accounts`, `b2b_accounts`, `md_accounts`, or simply `accounts`.
- ❌ `/db/orders`, `/db/projects` without prefix — same applies.

### Dual-view catalogs (one table, multiple representations)

Multiple catalogs can point to **one DB table** via `$_KAT['TABLES']` mapping — showing different fields and applying different filters:

- **Clients:** `ag_clients`, `b2b_clients`, `md_clients`, `md_contractors` → one table. Deleting in one deletes in all. Fields and `access_db` are separate per catalog.
- **Accounts/users:** `accounts`, `users`, `ag_accounts`, `b2b_accounts`, `md_accounts`, `integrations` → table `auth_pers`.
- **Financial transactions:** `ag_cashflows`, `ag_cashflows_prj`, `ag_bdr`, `ag_balans_report`, `ag_profit_and_loss` → table `ag_cashflows`.
- **Projects:** `ag_projects`, `ag_sales_report`, `ag_workload_report`, `trash` → table `ag_projects`.

When working with such catalogs:
- Reading the "full list" — typically via the canonical name (`ag_clients`, `auth_pers`)
- Creating in a module context (B2B/MD) — via the module-specific name (`b2b_clients`) — afteradd will populate module-specific fields
- Deletion = global (soft-delete `hidden=1` across the entire table)

---

## Available Catalogs

### Finance (AG Module)

| Catalog | Description |
|---------|-------------|
| `ag_cashflows` | Transactions (cash flow) |
| `ag_cashflows_prj` | Project payments |
| `ag_cashflows_clients` | Transactions by counterparty |
| `ag_transactions` | Transactions |
| `ag_accountant` | Payment invoices |
| `ag_contracts` | Contracts |
| `ag_clients` | Counterparties |
| `ag_companies` | Your legal entities |
| `ag_projects` | Projects |
| `ag_products` | Products |
| `ag_products_group` | Product groups |
| `ag_articles` | Cash flow items |
| `ag_cat_articles` | Item categories |
| `ag_fonds` | Funds |
| `ag_fonds_percent` | Fund distribution |
| `ag_flows` | Flows |
| `ag_settlement_accounts` | Settlement accounts |
| `ag_dds` | Cash flow statement |
| `ag_bdr` | Budget of income and expenses |
| `ag_balans_report` | Balance sheet |
| `ag_sales_report` | Sales report |
| `ag_workload_report` | Workload report |
| `ag_products_report` | Cash flow by product |
| `ag_debt_credit_report` | Counterparty debts |
| `ag_sumcash` | Payment calendar by legal entity |
| `ag_fonds_calendar` | Payment calendar by fund |
| `ag_project_to_types` | Budget and profitability by product |
| `ag_p2t_plan` | Monthly plan by product |
| `ag_article_plan` | Monthly plan by item |
| `ag_accounts` | Employees (AG) |
| `bank_exchanges` | Bank exports |
| `currency_rate` | Exchange rates |

### B2B (Trade, Orders)

| Catalog | Description |
|---------|-------------|
| `b2b_orders` | Orders |
| `b2b_basket` | Order contents |
| `b2b_order_items` | Ordered products |
| `b2b_clients` | Clients |
| `b2b_items` | Products |
| `b2b_cat_items` | Product groups |
| `b2b_nomenclature_group` | Product categories |
| `b2b_discounts` | Discounts and prices |
| `b2b_prices_items` | Special product prices |
| `b2b_delivery` | Delivery addresses |
| `b2b_routes` | Routes |
| `b2b_route_items` | Products by route |
| `b2b_available_categories` | Available product categories |
| `b2b_production_schedule` | Production schedule |
| `b2b_accounts` | B2B users |
| `b2b_news` | News (B2B portal) |
| `basket_connect` | Cart connections |

### Manufacturing (MD Module)

| Catalog | Description |
|---------|-------------|
| `md_project` | Order processing |
| `md_production` | Production |
| `md_project_product` | Products in a project |
| `md_project_batch` | Batches in products |
| `md_batches` | Batches |
| `md_sketches` | Sketches |
| `md_design` | 3D Designer |
| `md_wages` | Payroll calculation |
| `md_wages_actual` | Salaries |
| `md_min_wages` | Base salaries |
| `md_awards` | Bonuses and penalties |
| `md_coefficients` | Coefficients |
| `md_contractors` | Contractors |
| `md_clients` | Clients (MD) |
| `md_basket` | Order contents (MD) |
| `md_accounts` | Employees (MD) |
| `md_alloy` | Alloy types |
| `md_articles` | Report comments |
| `md_cat_articles` | Item categories (MD) |
| `md_reports` | Reports |
| `md_production_report` | Production report |
| `md_remarks` | Remarks |

### Time Tracking (TT Module)

| Catalog | Description |
|---------|-------------|
| `tt_tasks` | Tasks |
| `tt_projects` | Projects |
| `tt_worklogs` | Work logs |
| `tt_versions` | Versions |
| `tt_wiki` | Wiki |
| `tt_comments` | Comments |

### Warehouse (WH Module)

| Catalog | Description |
|---------|-------------|
| `wh_items` | Products |
| `wh_cat_items` | Product groups |
| `wh_stores` | Warehouses |
| `wh_movements` | Stock movements |
| `wh_nomenclature_group` | Product categories |
| `wh_warehouse_report` | Warehouse report |

### Field Services (VRN Module)

| Catalog | Description |
|---------|-------------|
| `vrn_projects` | Jobs |
| `vrn_type_work` | Job types |
| `vrn_tools` | Tools |
| `vrn_equipment` | Machinery and equipment |
| `vrn_materials` | Materials |

### CRM

| Catalog | Description |
|---------|-------------|
| `crm_contacts` | Clients |
| `crm_orders` | Orders |
| `crm_delivery` | Delivery |
| `crm_stores` | Warehouses |
| `crm_items` | Products |
| `crm_basket` | Order contents |
| `crm_feedback` | Support tickets |

### System / Service

| Catalog | Description |
|---------|-------------|
| `accounts` | Employees (primary) |
| `users` | Platform users |
| `contacts` | Contacts |
| `project_types` | Project types |
| `activity` | User activity log |
| `eventlogs` | Integration events |
| `integrations` | Integrations |
| `notifications` | Notifications |
| `dashboard_widgets` | Dashboard widgets |
| `dashboards` | Dashboards |
| `saved_filters` | Saved filters |
| `apps_storage` | App KV storage |
| `installed_apps` | Installed apps (for a specific group/tenant) |
| `marketplace` | All marketplace apps |
| `todo` | To-do |
| `remarks` | Remarks |
| `bookmarks` | Bookmarks |
| `favorites_menu` | Favorites menu |
| `profile_company` | Company profile |
| `coredb_spravochnik` | Reference directory |
| `new_elements` | New items |
| `trash` | Trash |
| `docs` | Documents (doctxt) |
| `catalog_rules` | Catalog rules (afterSave/beforeSave) |

---

## Marketplace Catalogs — In Detail

### `marketplace` — global app registry

Contains **all** apps published in the marketplace, regardless of whether they are installed by the user. This is a public catalog — like an App Store.

Key response fields:

| Field | Type | Description |
|-------|------|-------------|
| `alias` | string | Unique app identifier |
| `name` | string | App name |
| `anons` | string | Short description (from `config.json → description`) |
| `about` | string | Full description (from `config.json → about`) |
| `tags` | string | Comma-separated tags |
| `doc` | string | Icon filename — path: `/data/db/f_marketplace/{doc}` |
| `from_group` | string | Group ID of the app owner (developer) |
| `package` | string | Package name (from `config.json → package`) |
| `version` | string | App version |

```js
// List all marketplace apps
const resp = await App.fetch('/db/marketplace.json');
// resp.data[0].doc → icon filename
// Icon path: `/data/db/f_marketplace/${item.doc}`
```

### `installed_apps` — tenant's installed apps

Contains apps **installed by a specific group** (tenant). This is not a global list — each group only sees their own installed apps. Populated automatically by the platform when a user installs from the marketplace UI.

**Important**: `installed_apps` is the single source of truth for "is this app installed for this user?". Do not confuse with `marketplace` (all apps) or `dashboard_widgets` (dashboard widgets, even if of the marketplace app type).

Key fields:

| Field | Type | Description |
|-------|------|-------------|
| `alias` | string | Unique alias of the record |
| `app_id` | string | **Alias from `marketplace`** — primary link |
| `name` | string | Name (copied from marketplace) |
| `from_group` | string | ID of the group that installed the app |

```js
// Check if app with package="my-app" is installed
const installed = await App.fetchAll('/db/installed_apps.json');
const myApp = installed.data.find(a => a.app_id === 'MY_MARKETPLACE_ALIAS');
const isInstalled = !!myApp;

// If you need the marketplace alias — it's in app_id field
const marketplaceAlias = myApp?.app_id;
```

### App Lifecycle

```
marketplace          → installed_apps         → dashboard_widgets
(global registry)     (installed by tenant)     (placed on dashboard)
     ↑                       ↑                          ↑
 deploy via API        user clicked                user added
 /db/marketplace/{id}  "Install" in UI             widget to workspace
```

Relationships:
- `installed_apps.app_id` → `marketplace.alias` (FK: what's installed)
- `dashboard_widgets.options.package` → `config.json → package` (what's displayed on dashboard)
- `apps_storage.app_id` → `installed_apps.id` (KV storage for a specific installation)

### Notifications and Communications

| Catalog | Description |
|---------|-------------|
| `send_partners_notifications` | Partner newsletters |
| `send_segments` | Mailing segments |
| `send_tpl_emails` | Email templates |
| `mail_notifications_queue` | Email queue |
| `telegram_notifications` | Telegram message queue |
| `tpl_emails_orders` | Email templates for orders |
| `tpl_emails_persons` | Email templates for partners |

### AI and Logs

| Catalog | Description |
|---------|-------------|
| `deepseek_request` | DeepSeek requests |
| `log_deepseek` | DeepSeek logs |
| `log_bad_authorize` | Failed authorization logs |
| `log_billings` | Billing logs |
| `api` | API tokens |

---

## Example App Requests

### Get B2B Orders

```js
App.fetch('/db/b2b_orders.json').then(resp => {
    console.log(resp.data); // array of orders
});

// With status filter
App.fetch('/db/b2b_orders.json?form[status]=new').then(resp => { ... });
```

### Order Contents

```js
App.getRequestParams().then(resp => {
    const orderId = resp.data.itemId;
    App.fetchAll(`/db/b2b_basket.json?form[order_id]=${orderId}`).then(resp => {
        console.log(resp.data); // all order items
    });
});
```

### Create a Task in TT

```js
App.fetch('/db/tt_tasks/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Task from app',
        'form[project_id]': projectAlias,
        submit: 1
    }
});
```

### Send a Request to DeepSeek

```js
App.fetch('/db/deepseek_request/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[prompt]': 'Analyze the order',
        'form[context]': JSON.stringify(orderData),
        submit: 1
    }
});
```

### Add a Telegram Notification

```js
App.fetch('/db/telegram_notifications/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[message]': 'New order #' + orderId,
        'form[chat_id]': chatId,
        submit: 1
    }
});
```

### Read Activity History

```js
App.fetch('/db/activity.json?form[catalog]=b2b_orders&form[item_id]=ORDER_ALIAS')
    .then(resp => {
        resp.data.forEach(event => {
            console.log(event.user, event.action, event.date);
        });
    });
```

---

## End-to-End Automation Scenarios

Korfix is an ERP with a unified database, so an app can build
**end-to-end scenarios** across multiple modules without context switching.

### B2B Order → Manufacturing → Invoice

```js
App.getRequestParams().then(async resp => {
    const orderId = resp.data.itemId;

    // 1. Read the order
    const order = await App.fetch(`/db/b2b_orders/${orderId}`);

    // 2. Create a manufacturing project
    const mdProject = await App.fetch('/db/md_project/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[name]': 'Production ' + order.data.name,
            'form[b2b_order_id]': orderId,
            submit: 1
        }
    });

    // 3. Create a payment invoice
    await App.fetch('/db/ag_accountant/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[name]': 'Invoice for ' + order.data.name,
            'form[amount]': order.data.total,
            'form[client_id]': order.data.client_id,
            submit: 1
        }
    });

    App.alert('Project and invoice created', 'Done');
    App.reload();
});
```

### Webhook on New Orders → Telegram

```js
// Register once on app install
App.storage.set(
    'event.hook.activity.b2b_orders.добавил.json',
    'https://myapp.example.com/api/new-order-hook'
);
```

The app backend receives a POST with event data
and sends a message to Telegram via `telegram_notifications` or directly.

---

## Enabled Platform Features (conf.phtml)

These capabilities are active in Korfix — apps can rely on them:

| Feature | Value |
|---------|-------|
| `marketplace` | ✓ marketplace |
| `core_events` | ✓ event system (afteradd, beforesave, activity) |
| `access_db` | ✓ row-level catalog permissions |
| `access_statuses` | ✓ status permissions |
| `custom_dbfields` | ✓ custom fields in catalogs |
| `saved_filters` | ✓ saved filters |
| `currency_rate` | ✓ exchange rates |
| `json_response` | ✓ everything returns JSON |
| `cataccess` | ✓ row-level access |
| `partners` | ✓ B2B client authentication |
| `yookassa` | ✓ payment system |
| `bitrix24` | ✓ Bitrix24 synchronization |
| `smtp` | ✓ email sending |
| `multiauth` | ✓ multiple roles per user |
| `catalog_rules` | ✓ declarative afterSave/beforeSave rules |

---

## Catalog Relationships (FK)

Relationships are defined via `select_from_table` type fields in schemas. When requested via the API, related values are substituted automatically (the `{name}_value` field in the response).

### Constraint Types (WHERE)

Most relationships are restricted by visibility conditions:

| Pattern | Description |
|---------|-------------|
| `from_group = SESSION_GROUP` | Only records from own group (tenant isolation) |
| `$_ACCESS->get_where()` | Full permission check via the access system |
| `from_group = ... OR from_group = '0'` | Own + shared (template) records |
| `from_auth = SESSION_ID OR from_auth = 0` | Own + public (e.g. dashboards) |
| `hidden < 1 OR hidden IS NULL` | Not deleted |
| `account_type = N` | Filter by user role |

### Finance (AG)

```
ag_cashflows
  ├─ project_id    → ag_projects.id
  ├─ expense_type_id → ag_articles.id      (cash flow item)
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id       (own legal entity)
  ├─ settlement_account_id → ag_settlement_accounts.id
  ├─ from_auth     → auth_pers.author_id
  └─ members_id    → auth_pers.author_id[] (multi)

ag_contracts
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id
  ├─ project_id    → ag_projects.id[]      (multi)
  └─ members_id    → auth_pers.author_id[]

ag_articles
  ├─ category      → ag_cat_articles.id
  ├─ flows_id      → ag_flows.id
  ├─ products      → ag_products.id[]      (multi)
  └─ members_id    → auth_pers.author_id[]

ag_accountant (Invoices)
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id
  └─ members_id    → auth_pers.author_id[]

ag_projects
  ├─ type_project_id → ag_products.id      (project type)
  ├─ contact_id    → ag_clients.id
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id
  └─ members_id    → auth_pers.author_id[]

ag_products
  └─ group_id      → ag_products_group.id

ag_fonds_percent
  ├─ expense_types_id → ag_articles.id
  └─ fond_id       → ag_fonds.id

ag_clients / ag_companies
  └─ country_id    → country.alias
```

### B2B (Trade)

```
b2b_orders
  ├─ client_id     → b2b_clients.id
  ├─ delivery_id   → b2b_delivery.id
  └─ members_id    → auth_pers.author_id[]

b2b_basket (Order contents)
  ├─ order_id      → b2b_orders.id
  └─ item_id       → b2b_items.id

b2b_items (Products)
  ├─ nomenclature_group → b2b_cat_items.id (category)
  └─ members_id    → auth_pers.author_id[]

b2b_delivery
  └─ route_id      → b2b_routes.id (JOIN with b2b_production_schedule)

b2b_discounts
  └─ client_group  → b2b_clients.id
```

### Tasks (TT)

```
tt_tasks
  ├─ person_id     → auth_pers.author_id   (assignee)
  ├─ version_id    → tt_versions.id
  ├─ project_id    → tt_projects.id
  ├─ parent_id     → tt_tasks.id           (parent task)
  └─ members_id    → auth_pers.author_id[]

tt_worklogs
  ├─ person_id     → auth_pers.author_id
  ├─ task_id       → tt_tasks.id
  └─ members_id    → auth_pers.author_id[]

tt_wiki
  ├─ project_id    → tt_projects.id
  └─ members_id    → auth_pers.author_id[]

tt_comments
  ├─ task_id       → tt_tasks.id
  └─ from_auth     → auth_pers.author_id
```

### Manufacturing (MD)

```
md_project
  ├─ client_id     → ag_clients.id
  ├─ contract_id   → ag_contracts.id
  └─ members_id    → auth_pers.author_id[]

md_production
  └─ project_id    → md_project.id

md_project_batch
  └─ project_id    → md_project.id

md_project_product
  └─ project_id    → md_project.id
```

### Warehouse (WH)

```
wh_movements
  ├─ store_id      → wh_stores.id
  ├─ item_id       → wh_items.id
  └─ members_id    → auth_pers.author_id[]

wh_items
  ├─ category_id   → wh_categories.id
  └─ members_id    → auth_pers.author_id[]
```

### Field Services (VRN)

```
vrn_projects
  ├─ type_work_id  → vrn_type_work.id
  └─ members_id    → auth_pers.author_id[]
```

### CRM

```
crm_orders
  ├─ contact_id    → crm_contacts.id
  └─ members_id    → auth_pers.author_id[]

crm_basket
  ├─ order_id      → crm_orders.id
  └─ item_id       → crm_items.id
```

### Dashboards

```
dashboard_widgets
  ├─ board_id      → dashboards.id
  │    WHERE: (from_auth = SESSION_ID OR from_auth = 0) AND (hidden IS NULL OR hidden < 1)
  └─ from_auth     → auth_pers.author_id
```

### System

```
installed_apps
  └─ app_id        → marketplace.id

apps_storage
  └─ app_id        → installed_apps.id

custom_dbfields
  ├─ scheme        → custom_dbtables.dbname (catalog the field is attached to)
  └─ from_auth     → auth_pers.author_id

custom_dbview
  ├─ catalog1      → (any catalog)
  ├─ catalog2      → (any catalog)
  ├─ catalog1_field → (field from catalog1)
  └─ catalog2_field → (field from catalog2)
```

### Common members_id Pattern

Almost all business catalogs have a `members_id` field (multiselect_from_table → auth_pers).
These are participants/watchers of the record. WHERE constraint: `from_group = SESSION_GROUP AND hidden < 1`.

### Using Relationships in Miniapps

```js
// API returns related values when load_values=1
const resp = await App.fetch('/api/db/ag_cashflows?limit=10&load_values=1');
// resp.data[0].client_id = "5"
// resp.data[0].client_id_value = "Acme Corp"

// Or load the schema to get select options
const schema = await App.fetch('/db/ag_cashflows/sheme.json');
// schema.data.client_id.arr = {5: "Acme Corp", 12: "John Smith LLC"}
```

---

*Catalogs: /modules/db/ in panel.korfix.ru*

**← [Home](index.md)**
