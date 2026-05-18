# Marketplace — Miniapp Development

Documentation for developing marketplace applications (miniapps) for the Korfix / VMCRM platform.

> **Start here:** [rules.md](rules.md) — sandbox rules, restrictions, principles. **Read before starting.**
> Server architecture, core hooks, infrastructure — in [../backend/index.md](../backend/index.md)

---

## Quick Start

1. [rules.md](rules.md) — what is and isn't allowed (required)
2. [getting-started.md](getting-started.md) — your first app in 15 minutes
3. [deploy.md](deploy.md) — packaging and uploading via API
4. [checklist.md](checklist.md) — pre-release checklist

---

## By Topic

### Core API — communicating with the platform
| File | Description |
|------|-------------|
| [config-json.md](config-json.md) | Entry points, permissions, dashboard widgets |
| [js-api.md](js-api.md) | `VMCRMUserApp`: methods, events, CORS, navigation |
| [data-api.md](data-api.md) | Catalog CRUD, `form[]` vs plain, normalization, filters |
| [storage-and-hooks.md](storage-and-hooks.md) | `App.storage` (KV), webhooks, `afterSave` |

### UI and Styling
| File | Description |
|------|-------------|
| [styling.md](styling.md) | CSS variables, components, Chart.js, responsive, auto-resize |
| [dashboards.md](dashboards.md) | Dashboards and widgets: types, parameters, API creation |

### Catalog Reference (what data you can read)
| File | Description |
|------|-------------|
| [korfix-catalogs.md](korfix-catalogs.md) | All available Korfix ERP catalogs |
| [favorites-menu.md](favorites-menu.md) | `favorites_menu` — bookmarks and start page |
| [systempush-settings.md](systempush-settings.md) | `systempush_settings` — push subscriptions |
| [account-help.md](account-help.md) | `account_help`, `service_help` — contextual help |
| [access-statuses.md](access-statuses.md) | `access_statuses` — role-based status permissions |
| [bitrix24-sync.md](bitrix24-sync.md) | `bitrix24_sync` — two-way Bitrix24 synchronization |

### App Architecture
| File | Description |
|------|-------------|
| [frames.md](frames.md) | Standard frame types: `install`, `main`, `footer`, `widget` — conventions and patterns |

### Platform Features
| File | Description |
|------|-------------|
| [self-provisioning.md](self-provisioning.md) | Creating catalogs and fields on install |
| [catalog-rules.md](catalog-rules.md) | Declarative afterSave rules: inherit / calc / aggregate / validate |
| [catalog-settings.md](catalog-settings.md) | Catalog display settings (columns, order) |
| [db-views.md](db-views.md) | VIEW representations: joining catalogs via LEFT JOIN |

### Lifecycle — from code to release
| File | Description |
|------|-------------|
| [rules.md](rules.md) | Rules and restrictions |
| [getting-started.md](getting-started.md) | Your first app |
| [deploy.md](deploy.md) | Packaging, uploading, CI/CD |
| [checklist.md](checklist.md) | Pre-release checklist |

---

## Task → Which Files to Read

| I want to... | Files |
|--------------|-------|
| Embed a widget below a catalog list | [config-json.md](config-json.md) + [js-api.md](js-api.md) + [styling.md](styling.md) |
| Add a main menu item | [config-json.md](config-json.md) + [getting-started.md](getting-started.md) |
| Read/write catalog data | [data-api.md](data-api.md) + [js-api.md](js-api.md) |
| Create a custom catalog on install | [self-provisioning.md](self-provisioning.md) + [data-api.md](data-api.md) |
| Structure frames (install/main/widget) | [frames.md](frames.md) |
| React to record saves | [catalog-rules.md](catalog-rules.md) (declarative) **or** [storage-and-hooks.md](storage-and-hooks.md) (webhook) |
| Build a dashboard widget | [dashboards.md](dashboards.md) + [config-json.md](config-json.md) |
| Combine data from two catalogs | [db-views.md](db-views.md) |
| Work with push notifications | [systempush-settings.md](systempush-settings.md) |
| Sync with Bitrix24 | [bitrix24-sync.md](bitrix24-sync.md) |
| Send requests from a server (outside iframe) | [data-api.md](data-api.md) (Bearer tokens section) |
| Store app settings | [storage-and-hooks.md](storage-and-hooks.md) |
| Style the UI to match the platform | [styling.md](styling.md) |
| Deploy an app | [deploy.md](deploy.md) |
| Verify an app before release | [checklist.md](checklist.md) |

---

## Documentation Conventions

- Each file has a **"See also"** block at the top — related topics.
- At the bottom — **"Next: X"** for linear reading and **← INDEX** to return.
- Code examples are working — copy-paste should run.
- Reference and experimental miniapps — see the public repo [korfixdev/miniapps-examples](https://github.com/korfixdev) (when available) or ask the Korfix team.
