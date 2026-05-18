# Catalog Rules (catalog_rules)

> **See also:** [storage-and-hooks.md](storage-and-hooks.md) · [self-provisioning.md](self-provisioning.md) · [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

Declarative afterSave/beforeSave rules — automation without PHP code.

> Backend architecture: `lib/local/CatalogRules.php`

---

## Concept

A record in the `catalog_rules` catalog = one rule. When a record is saved in any catalog,
the core checks rules and executes them automatically: copying fields, recalculation, aggregation, validation.

Rules execute **after** PHP hooks (afteradd/beforesave) in `priority` order.

Access: root only (group creator).

---

## Record Fields

| Field | Description |
|-------|-------------|
| `name` | Rule name (for your reference) |
| `catalog` | Catalog the rule is attached to |
| `trigger_event` | `afterSave` or `beforeSave` |
| `rule_type` | `inherit`, `calc`, `aggregate`, `validate` |
| `config` | JSON configuration (see below) |
| `priority` | Execution order (lower = earlier, default 100) |
| `enabled` | Enabled/disabled |

---

## Rule Types and JSON Syntax

### 1. inherit — copy a field from a linked record

Copies a field value from the record referenced by an FK.

**When to use:** a field should be auto-populated from a related catalog.

**Minimal JSON:**

```json
{
    "source_field": "client_id",
    "source_catalog": "b2b_clients",
    "source_lookup": "acc_id",
    "target_field": "from_auth"
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `source_field` | yes | FK field of the current record (reference to another catalog) |
| `source_catalog` | yes | Table to read from (without `crm__` prefix) |
| `source_lookup` | yes | Field in source_catalog to copy |
| `target_field` | yes | Field of current record to write result into |
| `source_join` | no | Additional JOIN for complex relationships |

**Examples:**

Copy `from_auth` from client to order:
```json
{
    "source_field": "client_id",
    "source_catalog": "b2b_clients",
    "source_lookup": "acc_id",
    "target_field": "from_auth"
}
```

Copy discount percentage via JOIN (client → discount → percent):
```json
{
    "source_field": "client_id",
    "source_catalog": "b2b_clients",
    "source_join": "LEFT JOIN crm__b2b_discounts d ON d.id = crm__b2b_clients.discount_id",
    "source_lookup": "d.percent",
    "target_field": "percent"
}
```

**trigger_event:** `afterSave`

---

### 2. calc — compute a field from a formula

Computes a field value from other fields in the same record.

**When to use:** automatic calculation of totals, sums, percentages.

**Minimal JSON:**

```json
{
    "formula": "price * quantity",
    "target_field": "summa"
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `formula` | yes | Expression. Variables = field names of the current record |
| `target_field` | yes | Field to write the result into |

**Supported operations:**

| Operation | Example |
|-----------|---------|
| `+` `-` `*` `/` | `price * quantity` |
| `( )` | `(price - discount) * quantity` |
| `round()` | `round(price * 1.2)` |
| `ceil()` | `ceil(weight / 1000)` |
| `floor()` | `floor(total / 12)` |

If a field is not found — substituted with `0`. Division by 0 returns `0`.

**Examples:**

Line item total:
```json
{
    "formula": "price * quantity",
    "target_field": "summa"
}
```

Total with VAT:
```json
{
    "formula": "round(price * quantity * 1.2)",
    "target_field": "summa_vat"
}
```

Margin:
```json
{
    "formula": "revenue - cost",
    "target_field": "margin"
}
```

**trigger_event:** `afterSave`

---

### 3. aggregate — update a parent record

Runs an aggregate function over child records and writes the result to the parent.

**When to use:** recalculate order total when a cart item changes, count elements.

**Minimal JSON:**

```json
{
    "parent_field": "order_id",
    "parent_catalog": "b2b_orders",
    "function": "SUM",
    "source_field": "summa",
    "target_field": "summ"
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `parent_field` | yes | FK field of the current record (reference to parent) |
| `parent_catalog` | yes | Parent table (without prefix) |
| `function` | yes | `SUM`, `COUNT`, `AVG`, `MAX`, `MIN` |
| `source_field` | yes | Field of current catalog to aggregate |
| `target_field` | yes | Parent field to write result into |
| `where` | no | Additional filter (default: `hidden < 1 OR hidden IS NULL`) |

**Examples:**

Cart total → order:
```json
{
    "parent_field": "order_id",
    "parent_catalog": "b2b_orders",
    "function": "SUM",
    "source_field": "summa",
    "target_field": "summ"
}
```

Task count → project:
```json
{
    "parent_field": "project_id",
    "parent_catalog": "tt_projects",
    "function": "COUNT",
    "source_field": "id",
    "target_field": "tasks_count"
}
```

Latest date → project:
```json
{
    "parent_field": "project_id",
    "parent_catalog": "ag_projects",
    "function": "MAX",
    "source_field": "date_fact",
    "target_field": "last_payment_date",
    "where": "status = 30 AND (hidden < 1 OR hidden IS NULL)"
}
```

**trigger_event:** `afterSave`

---

### 4. validate — check a condition before saving

Checks a condition and blocks saving if it fails.

**When to use:** minimum order amount, required field for a specific status.

**Minimal JSON:**

```json
{
    "check": "summ >= 1000",
    "error_message": "Order total must be at least 1000"
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `condition` | no | Activation condition. If not met — rule is skipped |
| `check` | yes | Expression to verify. If false — error |
| `error_message` | yes | Error message. Supports `{field}` substitution |
| `revert_field` | no | Field to revert on error |
| `revert_value` | no | Value to revert to |

**Supported operators:** `==`, `!=`, `>`, `<`, `>=`, `<=`

**Examples:**

Minimum order total:
```json
{
    "condition": "status == 10",
    "check": "summ >= 1000",
    "error_message": "Order total ({summ}) is below the minimum (1000)",
    "revert_field": "status",
    "revert_value": "0"
}
```

Required field when status is "completed":
```json
{
    "check": "result != ",
    "condition": "status == 40",
    "error_message": "Provide a result before completing the task"
}
```

Prevent deletion of paid records:
```json
{
    "condition": "status == 30",
    "check": "hidden == 0",
    "error_message": "Cannot delete a paid record"
}
```

**trigger_event:** `beforeSave`

---

## Working with Rules via API (from a miniapp)

### Read

```js
const resp = await App.fetch('/db/catalog_rules.json');
resp.data.forEach(rule => {
    console.log(rule.catalog, rule.rule_type, JSON.parse(rule.config));
});

// Rules for a specific catalog
const resp = await App.fetch('/db/catalog_rules.json?form[catalog]=b2b_orders');
```

### Create

```js
await App.fetch('/db/catalog_rules/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Recalculate order total',
        'form[catalog]': 'b2b_basket',
        'form[trigger_event]': 'afterSave',
        'form[rule_type]': 'aggregate',
        'form[config]': JSON.stringify({
            parent_field: 'order_id',
            parent_catalog: 'b2b_orders',
            function: 'SUM',
            source_field: 'summa',
            target_field: 'summ'
        }),
        'form[priority]': 100,
        'form[enabled]': 1,
        submit: 1
    }
});
```

### Enable/Disable

```js
await App.fetch(`/db/catalog_rules/${alias}?edit&ajax=1`, {
    method: 'POST',
    body: { 'form[enabled]': 0, submit: 1 }
});
```

---

## Execution Order

1. PHP beforesave.inc.php (if present)
2. **catalog_rules** with `trigger_event = beforeSave` (validate) — by priority
3. `KAT::save()` — writes to DB
4. PHP afteradd.inc.php (if present)
5. **catalog_rules** with `trigger_event = afterSave` (inherit, calc, aggregate) — by priority

Rules for the same catalog execute in ascending `priority` order (default 100).
Rules of different types can be combined — e.g., inherit (priority=10),
then calc (priority=20), then aggregate (priority=30).

---

## Error Handling

| Type | On error |
|------|----------|
| inherit | Logs to `Main::log_message()`, record saves |
| calc | Logs, record saves |
| aggregate | Logs, record saves |
| validate | Shows error to user, reverts field if `revert_field` is set |

Errors are written to the log with type `catalog_rules`.

---

### 5. create — create a record in another catalog

Creates a new record in the specified catalog, mapping fields from the current record.

**When to use:** auto-create a task when an order is added, create a log entry on status change.

**Minimal JSON:**

```json
{
    "dest_catalog": "tt_tasks",
    "fields": [
        {"dest_field": "name", "type": "select", "src_field": "name"},
        {"dest_field": "status", "type": "value", "value": "0"}
    ]
}
```

| Parameter | Required | Description |
|-----------|----------|-------------|
| `dest_catalog` | yes | Catalog to create the record in (without prefix) |
| `condition` | no | Activation condition (e.g. `status == 10`) |
| `on_action` | no | Only on `INS` or `UPD`. Empty = both |
| `fields` | yes | Field mapping array (see below) |

**Field format:**

```json
{"dest_field": "name", "type": "select", "src_field": "title"}
```
`type=select` — copy value from field `src_field` of the current record.

```json
{"dest_field": "status", "type": "value", "value": "0"}
```
`type=value` — write a constant.

**Examples:**

Create a task on new order:
```json
{
    "dest_catalog": "tt_tasks",
    "on_action": "INS",
    "fields": [
        {"dest_field": "name", "type": "select", "src_field": "name"},
        {"dest_field": "project_id", "type": "value", "value": "5"},
        {"dest_field": "status", "type": "value", "value": "5"}
    ]
}
```

Create a remark on status change to "completed":
```json
{
    "dest_catalog": "remarks",
    "condition": "status == 40",
    "fields": [
        {"dest_field": "name", "type": "value", "value": "Auto-close"},
        {"dest_field": "entity_type", "type": "value", "value": "tt_tasks"},
        {"dest_field": "entity_id", "type": "select", "src_field": "id"}
    ]
}
```

**trigger_event:** `afterSave`

---

## Limitations

- calc formulas: arithmetic on the current record's fields only, no cross-catalog lookups
- validate conditions: single comparison (`field op value`), no AND/OR
- inherit does not create records — only copies values from existing ones
- aggregate only works with numeric fields
- Rules do not cascade: if inherit changed a field, calc in the same cycle won't see it (requires another save)

---

*Catalog: `/db/catalog_rules` | Class: `lib/local/CatalogRules.php`*

**← [Home](index.md)**
