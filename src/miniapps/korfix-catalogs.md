# Маркетплейс -- Каталоги Korfix

> **См. также:** [data-api.md](data-api.md) · [self-provisioning.md](self-provisioning.md) · [db-views.md](db-views.md) · [catalog-settings.md](catalog-settings.md)
> **← [Home](index.md)**

Korfix -- полноценная ERP-платформа. Все каталоги доступны из приложения
через `App.fetch('/db/{catalog}.json', ...)` с правами текущего пользователя.

---

## ⚠️ Имена каталогов — всегда с префиксом модуля

Все каталоги Korfix именуются с префиксом модуля: `ag_*`, `b2b_*`, `md_*`, `crm_*`, `wh_*`, `tt_*` и т.п.

**Generic-имён без префикса не существует:**

- ❌ `/db/clients` — нет такого каталога. Возможно имеется в виду `crm_clients` (общий CRM), `ag_clients` (контрагенты в финансах), `b2b_clients` (клиенты B2B) или `md_clients` (клиенты MD). **Уточни у пользователя какой именно** — не угадывай.
- ❌ `/db/users` — нет. Доступны `auth_pers` (все аккаунты), `ag_accounts`, `b2b_accounts`, `md_accounts` или просто `accounts`.
- ❌ `/db/orders`, `/db/projects` без префикса — то же самое.

### Двойственные каталоги (одна таблица, разные представления)

Несколько каталогов могут смотреть в **одну БД-таблицу** через `$_KAT['TABLES']`-маппинг — показывают разные поля и применяют разные фильтры:

- **Клиенты:** `ag_clients`, `b2b_clients`, `md_clients`, `md_contractors` → одна таблица. Удалил в одном — пропал во всех. Поля и `access_db` — раздельные per каталог.
- **Аккаунты/пользователи:** `accounts`, `users`, `ag_accounts`, `b2b_accounts`, `md_accounts`, `integrations` → таблица `auth_pers`.
- **Финансовые операции:** `ag_cashflows`, `ag_cashflows_prj`, `ag_bdr`, `ag_balans_report`, `ag_profit_and_loss` → таблица `ag_cashflows`.
- **Проекты:** `ag_projects`, `ag_sales_report`, `ag_workload_report`, `trash` → таблица `ag_projects`.

При работе с такими каталогами:
- Чтение «полного списка» — обычно через каноничный (`ag_clients`, `auth_pers`)
- Создание в контексте конкретного модуля (B2B/MD) — через модульный (`b2b_clients`) — afteradd допишет специфичные поля
- Удаление = глобальное (soft-delete `hidden=1` во всей таблице)

---

## Доступные каталоги

### Финансы (AG-модуль)

| Каталог | Описание |
|---------|----------|
| `ag_cashflows` | Операции (ДДС) |
| `ag_cashflows_prj` | Выплаты по проектам |
| `ag_cashflows_clients` | Операции по контрагентам |
| `ag_transactions` | Транзакции |
| `ag_accountant` | Счета на оплату |
| `ag_contracts` | Договоры |
| `ag_clients` | Контрагенты |
| `ag_companies` | Ваши юрлица |
| `ag_projects` | Проекты |
| `ag_products` | Продукты |
| `ag_products_group` | Группы продуктов |
| `ag_articles` | Статьи ДДС |
| `ag_cat_articles` | Категории статей |
| `ag_fonds` | Фонды |
| `ag_fonds_percent` | Распределение по фондам |
| `ag_flows` | Потоки |
| `ag_settlement_accounts` | Расчётные счета |
| `ag_dds` | ДДС (отчёт) |
| `ag_bdr` | Бюджет доходов и расходов |
| `ag_balans_report` | Баланс |
| `ag_sales_report` | Отчёт по продажам |
| `ag_workload_report` | Отчёт по занятости |
| `ag_products_report` | ДДС по продуктам |
| `ag_debt_credit_report` | Задолженности контрагентов |
| `ag_sumcash` | Платёжный календарь по юрлицам |
| `ag_fonds_calendar` | Платёжный календарь по фондам |
| `ag_project_to_types` | Бюджет и рентабельность по продуктам |
| `ag_p2t_plan` | Месячный план по продуктам |
| `ag_article_plan` | Месячный план по статьям |
| `ag_accounts` | Сотрудники (AG) |
| `bank_exchanges` | Банковские выгрузки |
| `currency_rate` | Курсы валют |

### B2B (торговля, заказы)

| Каталог | Описание |
|---------|----------|
| `b2b_orders` | Заказы |
| `b2b_basket` | Состав заказа |
| `b2b_order_items` | Заказанная продукция |
| `b2b_clients` | Клиенты |
| `b2b_items` | Продукция |
| `b2b_cat_items` | Группы продукции |
| `b2b_nomenclature_group` | Категории продукции |
| `b2b_discounts` | Скидки и цены |
| `b2b_prices_items` | Особые цены на продукцию |
| `b2b_delivery` | Адреса доставки |
| `b2b_routes` | Маршруты |
| `b2b_route_items` | Продукция по маршрутам |
| `b2b_available_categories` | Доступные категории продуктов |
| `b2b_production_schedule` | График производства |
| `b2b_accounts` | Пользователи B2B |
| `b2b_news` | Новости (B2B портал) |
| `basket_connect` | Связь корзин |

### Производство (MD-модуль)

| Каталог | Описание |
|---------|----------|
| `md_project` | Обработка заказов |
| `md_production` | Производство |
| `md_project_product` | Изделия в проекте |
| `md_project_batch` | Партии в изделиях |
| `md_batches` | Партии |
| `md_sketches` | Эскизы |
| `md_design` | 3D Модельер |
| `md_wages` | Расчёт ЗП |
| `md_wages_actual` | Зарплаты |
| `md_min_wages` | Оклады |
| `md_awards` | Премии и штрафы |
| `md_coefficients` | Коэффициенты |
| `md_contractors` | Подрядчики |
| `md_clients` | Клиенты (MD) |
| `md_basket` | Состав заказов (MD) |
| `md_accounts` | Сотрудники (MD) |
| `md_alloy` | Виды сплавов |
| `md_articles` | Комментарии к отчёту |
| `md_cat_articles` | Категории статей (MD) |
| `md_reports` | Отчёты |
| `md_production_report` | Отчёт производства |
| `md_remarks` | Комментарии |

### Time Tracking (TT-модуль)

| Каталог | Описание |
|---------|----------|
| `tt_tasks` | Задачи |
| `tt_projects` | Проекты |
| `tt_worklogs` | Рабочие журналы |
| `tt_versions` | Версии |
| `tt_wiki` | Wiki |
| `tt_comments` | Комментарии |

### Склад (WH-модуль)

| Каталог | Описание |
|---------|----------|
| `wh_items` | Товары |
| `wh_cat_items` | Группы товаров |
| `wh_stores` | Склады |
| `wh_movements` | Движение товаров |
| `wh_nomenclature_group` | Категории товаров |
| `wh_warehouse_report` | Отчёт по складам |

### Выездные работы (VRN-модуль)

| Каталог | Описание |
|---------|----------|
| `vrn_projects` | Работы |
| `vrn_type_work` | Виды работ |
| `vrn_tools` | Инструменты |
| `vrn_equipment` | Техника и оборудование |
| `vrn_materials` | Материалы |

### CRM

| Каталог | Описание |
|---------|----------|
| `crm_contacts` | Клиенты |
| `crm_orders` | Заказы |
| `crm_delivery` | Доставка |
| `crm_stores` | Склады |
| `crm_items` | Товары |
| `crm_basket` | Состав заказов |
| `crm_feedback` | Обращения |

### Системные / служебные

| Каталог | Описание |
|---------|----------|
| `accounts` | Сотрудники (основной) |
| `users` | Пользователи платформы |
| `contacts` | Контакты |
| `project_types` | Типы проектов |
| `activity` | Лог активности пользователей |
| `eventlogs` | События интеграций |
| `integrations` | Интеграции |
| `notifications` | Уведомления |
| `dashboard_widgets` | Виджеты дашборда |
| `dashboards` | Рабочие столы |
| `saved_filters` | Сохранённые фильтры |
| `apps_storage` | KV-хранилище приложений |
| `installed_apps` | Установленные приложения |
| `marketplace` | Маркетплейс |
| `todo` | ToDo |
| `remarks` | Замечания |
| `bookmarks` | Закладки |
| `favorites_menu` | Избранное меню |
| `profile_company` | Профиль компании |
| `coredb_spravochnik` | Справочник |
| `new_elements` | Новые элементы |
| `trash` | Корзина |
| `docs` | Документы (doctxt) |
| `catalog_rules` | Правила каталогов (afterSave/beforeSave) |

### Рассылки и коммуникации

| Каталог | Описание |
|---------|----------|
| `send_partners_notifications` | Рассылки партнёрам |
| `send_segments` | Сегменты рассылки |
| `send_tpl_emails` | Шаблоны писем |
| `mail_notifications_queue` | Очередь писем |
| `telegram_notifications` | Очередь Telegram сообщений |
| `tpl_emails_orders` | Шаблоны писем по заказам |
| `tpl_emails_persons` | Шаблоны писем партнёрам |

### AI и логи

| Каталог | Описание |
|---------|----------|
| `deepseek_request` | Запросы к DeepSeek |
| `log_deepseek` | Логи DeepSeek |
| `log_bad_authorize` | Логи неудачных авторизаций |
| `log_billings` | Логи биллинга |
| `api` | API-токены |

---

## Примеры запросов из приложения

### Получить заказы B2B

```js
App.fetch('/db/b2b_orders.json').then(resp => {
    console.log(resp.data); // массив заказов
});

// С фильтром по статусу
App.fetch('/db/b2b_orders.json?form[status]=new').then(resp => { ... });
```

### Состав конкретного заказа

```js
App.getRequestParams().then(resp => {
    const orderId = resp.data.itemId;
    App.fetchAll(`/db/b2b_basket.json?form[order_id]=${orderId}`).then(resp => {
        console.log(resp.data); // все позиции заказа
    });
});
```

### Создать задачу в TT

```js
App.fetch('/db/tt_tasks/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[name]': 'Задача из приложения',
        'form[project_id]': projectAlias,
        submit: 1
    }
});
```

### Отправить запрос в DeepSeek

```js
App.fetch('/db/deepseek_request/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[prompt]': 'Проанализируй заказ',
        'form[context]': JSON.stringify(orderData),
        submit: 1
    }
});
```

### Добавить уведомление в Telegram

```js
App.fetch('/db/telegram_notifications/add?edit&ajax=1', {
    method: 'POST',
    body: {
        'form[message]': 'Новый заказ #' + orderId,
        'form[chat_id]': chatId,
        submit: 1
    }
});
```

### Прочитать историю активности

```js
App.fetch('/db/activity.json?form[catalog]=b2b_orders&form[item_id]=ORDER_ALIAS')
    .then(resp => {
        resp.data.forEach(event => {
            console.log(event.user, event.action, event.date);
        });
    });
```

---

## Сценарии сквозной автоматизации

Korfix -- ERP с единой базой, поэтому приложение может строить
**сквозные сценарии** через несколько модулей без переключения контекста.

### B2B Заказ → Производство → Счёт

```js
App.getRequestParams().then(async resp => {
    const orderId = resp.data.itemId;

    // 1. Читаем заказ
    const order = await App.fetch(`/db/b2b_orders/${orderId}`);

    // 2. Создаём производственный проект
    const mdProject = await App.fetch('/db/md_project/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[name]': 'Производство ' + order.data.name,
            'form[b2b_order_id]': orderId,
            submit: 1
        }
    });

    // 3. Создаём счёт на оплату
    await App.fetch('/db/ag_accountant/add?edit&ajax=1', {
        method: 'POST',
        body: {
            'form[name]': 'Счёт по заказу ' + order.data.name,
            'form[amount]': order.data.total,
            'form[client_id]': order.data.client_id,
            submit: 1
        }
    });

    App.alert('Проект и счёт созданы', 'Готово');
    App.reload();
});
```

### Вебхук на новые заказы → Telegram

```js
// Регистрируем один раз при установке приложения
App.storage.set(
    'event.hook.activity.b2b_orders.добавил.json',
    'https://myapp.example.com/api/new-order-hook'
);
```

На стороне бэкенда приложение получает POST с данными события
и шлёт сообщение в Telegram через `telegram_notifications` или напрямую.

---

## Включённые фичи платформы (conf.phtml)

Эти возможности активны в Korfix -- приложения могут на них опираться:

| Фича | Значение |
|------|----------|
| `marketplace` | ✓ маркетплейс |
| `core_events` | ✓ система событий (afteradd, beforesave, activity) |
| `access_db` | ✓ row-level права на каталоги |
| `access_statuses` | ✓ права на статусы |
| `custom_dbfields` | ✓ кастомные поля в каталогах |
| `saved_filters` | ✓ сохранённые фильтры |
| `currency_rate` | ✓ курсы валют |
| `json_response` | ✓ всё возвращает JSON |
| `cataccess` | ✓ права на уровне строк |
| `partners` | ✓ авторизация B2B-клиентов |
| `yookassa` | ✓ платёжная система |
| `bitrix24` | ✓ синхронизация с Bitrix24 |
| `smtp` | ✓ отправка email |
| `multiauth` | ✓ несколько ролей у пользователя |
| `catalog_rules` | ✓ декларативные правила afterSave/beforeSave |

---

## Связи между каталогами (FK)

Связи определены через поля типа `select_from_table` в схемах. При запросе через API связанные значения подставляются автоматически (поле `{name}_value` в ответе).

### Типы ограничений (WHERE)

Большинство связей ограничены условиями видимости:

| Паттерн | Описание |
|---------|----------|
| `from_group = SESSION_GROUP` | Только записи своей группы (тенант-изоляция) |
| `$_ACCESS->get_where()` | Полная проверка прав через систему доступа |
| `from_group = ... OR from_group = '0'` | Свои + общие (шаблонные) записи |
| `from_auth = SESSION_ID OR from_auth = 0` | Свои + публичные (напр. дашборды) |
| `hidden < 1 OR hidden IS NULL` | Не удалённые |
| `account_type = N` | Фильтр по роли пользователя |

### Финансы (AG)

```
ag_cashflows
  ├─ project_id    → ag_projects.id
  ├─ expense_type_id → ag_articles.id      (статья ДДС)
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id       (наше юрлицо)
  ├─ settlement_account_id → ag_settlement_accounts.id
  ├─ from_auth     → auth_pers.author_id
  └─ members_id    → auth_pers.author_id[] (мульти)

ag_contracts
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id
  ├─ project_id    → ag_projects.id[]      (мульти)
  └─ members_id    → auth_pers.author_id[]

ag_articles
  ├─ category      → ag_cat_articles.id
  ├─ flows_id      → ag_flows.id
  ├─ products      → ag_products.id[]      (мульти)
  └─ members_id    → auth_pers.author_id[]

ag_accountant (Счета)
  ├─ client_id     → ag_clients.id
  ├─ companies_id  → ag_companies.id
  └─ members_id    → auth_pers.author_id[]

ag_projects
  ├─ type_project_id → ag_products.id      (тип проекта)
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

### B2B (Торговля)

```
b2b_orders
  ├─ client_id     → b2b_clients.id
  ├─ delivery_id   → b2b_delivery.id
  └─ members_id    → auth_pers.author_id[]

b2b_basket (Состав заказа)
  ├─ order_id      → b2b_orders.id
  └─ item_id       → b2b_items.id

b2b_items (Продукция)
  ├─ nomenclature_group → b2b_cat_items.id (категория)
  └─ members_id    → auth_pers.author_id[]

b2b_delivery
  └─ route_id      → b2b_routes.id (JOIN с b2b_production_schedule)

b2b_discounts
  └─ client_group  → b2b_clients.id
```

### Задачи (TT)

```
tt_tasks
  ├─ person_id     → auth_pers.author_id   (исполнитель)
  ├─ version_id    → tt_versions.id
  ├─ project_id    → tt_projects.id
  ├─ parent_id     → tt_tasks.id           (родительская задача)
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

### Производство (MD)

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

### Склад (WH)

```
wh_movements
  ├─ store_id      → wh_stores.id
  ├─ item_id       → wh_items.id
  └─ members_id    → auth_pers.author_id[]

wh_items
  ├─ category_id   → wh_categories.id
  └─ members_id    → auth_pers.author_id[]
```

### Выездные работы (VRN)

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

### Дашборды

```
dashboard_widgets
  ├─ board_id      → dashboards.id
  │    WHERE: (from_auth = SESSION_ID OR from_auth = 0) AND (hidden IS NULL OR hidden < 1)
  └─ from_auth     → auth_pers.author_id
```

### Системные

```
installed_apps
  └─ app_id        → marketplace.id

apps_storage
  └─ app_id        → installed_apps.id

custom_dbfields
  ├─ scheme        → custom_dbtables.dbname (каталог, к которому привязано поле)
  └─ from_auth     → auth_pers.author_id

custom_dbview
  ├─ catalog1      → (любой каталог)
  ├─ catalog2      → (любой каталог)
  ├─ catalog1_field → (поле catalog1)
  └─ catalog2_field → (поле catalog2)
```

### Общий паттерн members_id

Почти все бизнес-каталоги имеют поле `members_id` (multiselect_from_table → auth_pers).
Это участники/наблюдатели записи. WHERE ограничение: `from_group = SESSION_GROUP AND hidden < 1`.

### Использование связей в миниапах

```js
// API возвращает связанные значения если load_values=1
const resp = await App.fetch('/api/db/ag_cashflows?limit=10&load_values=1');
// resp.data[0].client_id = "5"
// resp.data[0].client_id_value = "ООО Ромашка"

// Или загрузить схему для получения вариантов select
const schema = await App.fetch('/db/ag_cashflows/sheme.json');
// schema.data.client_id.arr = {5: "ООО Ромашка", 12: "ИП Иванов"}
```

---

*Каталоги: /modules/db/ в panel.korfix.ru*

**← [Home](index.md)**
