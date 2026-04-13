# Правила каталогов (catalog_rules)

> **См. также:** [storage-and-hooks.md](storage-and-hooks.md) · [self-provisioning.md](self-provisioning.md) · [data-api.md](data-api.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

Декларативные правила afterSave/beforeSave — автоматизация без PHP-кода.

> Бэкенд-архитектура: `lib/local/CatalogRules.php`

---

## Принцип

Запись в каталоге `catalog_rules` = одно правило. При сохранении записи в любом каталоге
ядро проверяет правила и выполняет их автоматически: копирование полей, пересчёт, агрегация, валидация.

Правила выполняются **после** PHP-хуков (afteradd/beforesave) по порядку `priority`.

Доступ: только root (создатель группы).

---

## Поля записи

| Поле | Описание |
|------|----------|
| `name` | Название правила (для себя) |
| `catalog` | Каталог, к которому привязано правило |
| `trigger_event` | `afterSave` или `beforeSave` |
| `rule_type` | `inherit`, `calc`, `aggregate`, `validate` |
| `config` | JSON-конфигурация правила (см. ниже) |
| `priority` | Порядок выполнения (меньше = раньше, по умолчанию 100) |
| `enabled` | Включено/выключено |

---

## Типы правил и JSON-синтаксис

### 1. inherit — наследование поля из связанной записи

Копирует значение поля из записи, на которую ссылается FK.

**Когда использовать:** поле должно автоматически заполняться из связанного каталога.

**Минимальный JSON:**

```json
{
    "source_field": "client_id",
    "source_catalog": "b2b_clients",
    "source_lookup": "acc_id",
    "target_field": "from_auth"
}
```

| Параметр | Обязательный | Описание |
|----------|-------------|----------|
| `source_field` | да | FK-поле текущей записи (ссылка на другой каталог) |
| `source_catalog` | да | Таблица, откуда брать значение (без префикса `crm__`) |
| `source_lookup` | да | Поле в source_catalog для копирования |
| `target_field` | да | Поле текущей записи для записи результата |
| `source_join` | нет | Дополнительный JOIN для сложных связей |

**Примеры:**

Скопировать `from_auth` из клиента в заказ:
```json
{
    "source_field": "client_id",
    "source_catalog": "b2b_clients",
    "source_lookup": "acc_id",
    "target_field": "from_auth"
}
```

Скопировать процент скидки через JOIN (клиент → скидка → процент):
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

### 2. calc — вычисление поля по формуле

Вычисляет значение поля из других полей той же записи.

**Когда использовать:** автоматический расчёт суммы, итога, процента.

**Минимальный JSON:**

```json
{
    "formula": "price * quantity",
    "target_field": "summa"
}
```

| Параметр | Обязательный | Описание |
|----------|-------------|----------|
| `formula` | да | Выражение. Переменные = имена полей текущей записи |
| `target_field` | да | Поле для записи результата |

**Поддерживаемые операции:**

| Операция | Пример |
|----------|--------|
| `+` `-` `*` `/` | `price * quantity` |
| `( )` | `(price - discount) * quantity` |
| `round()` | `round(price * 1.2)` |
| `ceil()` | `ceil(weight / 1000)` |
| `floor()` | `floor(total / 12)` |

Если поле не найдено — подставляется `0`. Деление на 0 возвращает `0`.

**Примеры:**

Сумма позиции:
```json
{
    "formula": "price * quantity",
    "target_field": "summa"
}
```

Сумма с НДС:
```json
{
    "formula": "round(price * quantity * 1.2)",
    "target_field": "summa_vat"
}
```

Маржа:
```json
{
    "formula": "revenue - cost",
    "target_field": "margin"
}
```

**trigger_event:** `afterSave`

---

### 3. aggregate — обновление родительской записи

Выполняет агрегатную функцию по дочерним записям и записывает результат в родителя.

**Когда использовать:** пересчёт суммы заказа при изменении корзины, подсчёт количества элементов.

**Минимальный JSON:**

```json
{
    "parent_field": "order_id",
    "parent_catalog": "b2b_orders",
    "function": "SUM",
    "source_field": "summa",
    "target_field": "summ"
}
```

| Параметр | Обязательный | Описание |
|----------|-------------|----------|
| `parent_field` | да | FK-поле текущей записи (ссылка на родителя) |
| `parent_catalog` | да | Таблица родителя (без префикса) |
| `function` | да | `SUM`, `COUNT`, `AVG`, `MAX`, `MIN` |
| `source_field` | да | Поле текущего каталога для агрегации |
| `target_field` | да | Поле родителя для записи результата |
| `where` | нет | Дополнительный фильтр (по умолчанию `hidden < 1 OR hidden IS NULL`) |

**Примеры:**

Сумма корзины → заказ:
```json
{
    "parent_field": "order_id",
    "parent_catalog": "b2b_orders",
    "function": "SUM",
    "source_field": "summa",
    "target_field": "summ"
}
```

Количество задач → проект:
```json
{
    "parent_field": "project_id",
    "parent_catalog": "tt_projects",
    "function": "COUNT",
    "source_field": "id",
    "target_field": "tasks_count"
}
```

Максимальная дата → проект:
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

### 4. validate — проверка условия перед сохранением

Проверяет условие и блокирует сохранение если оно не выполнено.

**Когда использовать:** минимальная сумма заказа, обязательность поля при определённом статусе.

**Минимальный JSON:**

```json
{
    "check": "summ >= 1000",
    "error_message": "Сумма заказа должна быть не менее 1000"
}
```

| Параметр | Обязательный | Описание |
|----------|-------------|----------|
| `condition` | нет | Условие активации правила. Если не выполнено — правило пропускается |
| `check` | да | Проверяемое выражение. Если false — ошибка |
| `error_message` | да | Текст ошибки. Поддерживает `{field}` подстановку |
| `revert_field` | нет | Поле для отката значения при ошибке |
| `revert_value` | нет | Значение для отката |

**Поддерживаемые операторы:** `==`, `!=`, `>`, `<`, `>=`, `<=`

**Примеры:**

Минимальная сумма заказа:
```json
{
    "condition": "status == 10",
    "check": "summ >= 1000",
    "error_message": "Сумма заказа ({summ}) меньше минимальной (1000)",
    "revert_field": "status",
    "revert_value": "0"
}
```

Обязательное поле при статусе "завершён":
```json
{
    "check": "result != ",
    "condition": "status == 40",
    "error_message": "Укажите результат перед завершением задачи"
}
```

Запрет удаления оплаченных записей:
```json
{
    "condition": "status == 30",
    "check": "hidden == 0",
    "error_message": "Нельзя удалить оплаченную запись"
}
```

**trigger_event:** `beforeSave`

---

## Работа с правилами через API (из миниапа)

### Чтение

```js
const resp = await App.fetch('/db/catalog_rules.json');
resp.data.forEach(rule => {
    console.log(rule.catalog, rule.rule_type, JSON.parse(rule.config));
});

// Правила конкретного каталога
const resp = await App.fetch('/db/catalog_rules.json?form[catalog]=b2b_orders');
```

### Создание

```js
await App.fetch('/db/catalog_rules/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Пересчёт суммы заказа',
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

### Включение/выключение

```js
await App.fetch(`/db/catalog_rules/${alias}?edit&ajax=1`, {
    method: 'POST',
    body: { 'form[enabled]': 0, submit: 1 }
});
```

---

## Порядок выполнения

1. PHP beforesave.inc.php (если есть)
2. **catalog_rules** с `trigger_event = beforeSave` (validate) — по priority
3. `KAT::save()` — запись в БД
4. PHP afteradd.inc.php (если есть)
5. **catalog_rules** с `trigger_event = afterSave` (inherit, calc, aggregate) — по priority

Правила одного каталога выполняются по возрастанию `priority` (100 по умолчанию).
Правила разных типов можно комбинировать — например сначала inherit (priority=10),
потом calc (priority=20), потом aggregate (priority=30).

---

## Обработка ошибок

| Тип | При ошибке |
|-----|-----------|
| inherit | Логирует в `Main::log_message()`, запись сохраняется |
| calc | Логирует, запись сохраняется |
| aggregate | Логирует, запись сохраняется |
| validate | Показывает ошибку пользователю, откатывает поле если указан `revert_field` |

Ошибки пишутся в лог с типом `catalog_rules`.

---

### 5. create — создание записи в другом каталоге

Создаёт новую запись в указанном каталоге, маппируя поля из текущей записи.

**Когда использовать:** автоматическое создание задачи при добавлении заказа, создание лога при смене статуса.

**Минимальный JSON:**

```json
{
    "dest_catalog": "tt_tasks",
    "fields": [
        {"dest_field": "name", "type": "select", "src_field": "name"},
        {"dest_field": "status", "type": "value", "value": "0"}
    ]
}
```

| Параметр | Обязательный | Описание |
|----------|-------------|----------|
| `dest_catalog` | да | Каталог для создания записи (без префикса) |
| `condition` | нет | Условие активации (например `status == 10`) |
| `on_action` | нет | Только при `INS` или `UPD`. Пусто = оба |
| `fields` | да | Массив маппинга полей (см. ниже) |

**Формат полей:**

```json
{"dest_field": "name", "type": "select", "src_field": "title"}
```
`type=select` — скопировать значение из поля `src_field` текущей записи.

```json
{"dest_field": "status", "type": "value", "value": "0"}
```
`type=value` — записать константу.

**Примеры:**

Создать задачу при новом заказе:
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

Создать замечание при смене статуса на "завершён":
```json
{
    "dest_catalog": "remarks",
    "condition": "status == 40",
    "fields": [
        {"dest_field": "name", "type": "value", "value": "Автозакрытие"},
        {"dest_field": "entity_type", "type": "value", "value": "tt_tasks"},
        {"dest_field": "entity_id", "type": "select", "src_field": "id"}
    ]
}
```

**trigger_event:** `afterSave`

---

## Ограничения

- Формулы calc: только арифметика с полями текущей записи, без обращения к другим каталогам
- Условия validate: только одно сравнение (`field op value`), без AND/OR
- Inherit не создаёт записи — только копирует значения из существующих
- Aggregate работает только с числовыми полями
- Правила не каскадируются: если inherit изменил поле, calc в том же цикле его не увидит (нужен следующий save)

---

*Каталог: `/db/catalog_rules` | Класс: `lib/local/CatalogRules.php`*

**← [Home](index.md)**
