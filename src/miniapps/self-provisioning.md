# Self-provisioning: создание структур данных

> **См. также:** [data-api.md](data-api.md) · [catalog-rules.md](catalog-rules.md) · [catalog-settings.md](catalog-settings.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

Приложение может создавать себе каталоги (таблицы) и кастомные поля при установке или первом запуске.

---

### Встроенный инсталлер (рекомендуемый)

Приложение при первом запуске проверяет наличие каталога и предлагает кнопку установки.
Не требует токенов или доступа к терминалу — работает через сессионную авторизацию (`App.fetch()`).

#### Проверка существования каталога

**Важно**: нельзя проверять существование кастомного каталога через `/db/{catalog}.json` —
при несуществующем каталоге CRM не возвращает ошибку, а fallback'ит на дефолтный каталог
(возвращает данные другого каталога со `status: "ok"`). Приложение ошибочно решит что каталог
существует и не покажет экран установки.

**Правильный способ** — проверять через реестр `custom_dbtables`:

```js
const CATALOG = 'custom_quicknotes';

async function checkCatalogExists() {
    try {
        // dbname = имя таблицы без префикса custom_ (quicknotes, не custom_quicknotes)
        const resp = await App.fetch('/db/custom_dbtables.json?form[dbname]=quicknotes');
        return !!(resp && resp.data && Array.isArray(resp.data) && resp.data.length > 0);
    } catch (e) {}
    return false;
}
```

Для приложений с несколькими каталогами — универсальная функция:

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

#### Создание каталога и полей

```js
const FIELDS = [
    { name: 'Текст заметки',  dbname: 'content',   type: 'textarea', f_maxlen: 65535 },
    { name: 'Приоритет',      dbname: 'priority',   type: 'select',   f_arr: 'обычный\nважный\nсрочный', f_default: 'обычный' },
    { name: 'Статус',         dbname: 'status',     type: 'select',   f_arr: 'активная\nвыполнена\nархив', f_default: 'активная' },
    { name: 'Дедлайн',        dbname: 'due_date',   type: 'datetime' },
];

// Получить ID текущего пользователя (для from_auth/from_group)
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

    // ОБЯЗАТЕЛЬНО: настроить права в access_db, иначе каталог невидим обычным ролям.
    // Default = 2 (self — каждый видит только свои записи) — типовой best-default.
    // Для коллаборативных каталогов (общие задачи, клиенты) — передать 1.
    // Функция configureAccess определена в разделе "Права доступа (access_db)" ниже.
    await configureAccess(CATALOG);  // CATALOG = 'custom_quicknotes'
}
```

> Без `configureAccess` после `createTable` каталог будет виден только админам — обычные роли получат пустой `data: []` (см. раздел «Права доступа (access_db)» ниже).

#### Полный паттерн: HTML с экраном установки

```html
<!-- Экран установки (скрыт по умолчанию) -->
<div id="installScreen" style="display:none;">
    <h2>Название приложения</h2>
    <p>Для работы нужен каталог данных. Нажмите кнопку для установки.</p>
    <button id="btnInstall">Установить структуру данных</button>
    <div id="installLog"></div>
</div>

<!-- Рабочий UI (скрыт по умолчанию) -->
<div id="mainUI" style="display:none;">
    <!-- ... основной интерфейс приложения ... -->
</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

async function init() {
    const exists = await checkCatalogExists();
    if (exists) {
        document.getElementById('mainUI').style.display = '';
        loadData();
    } else {
        document.getElementById('installScreen').style.display = '';
    }
}

document.getElementById('btnInstall').addEventListener('click', async () => {
    const btn = document.getElementById('btnInstall');
    btn.disabled = true;
    btn.textContent = 'Установка...';

    try {
        await runInstall();
        document.getElementById('installScreen').style.display = 'none';
        document.getElementById('mainUI').style.display = '';
        loadData();
    } catch (e) {
        btn.disabled = false;
        btn.textContent = 'Повторить установку';
    }
});

init();
</script>
```

### Внешний скрипт (curl / CI/CD)

Создание структуры через терминал с bearer-токеном:

```bash
# Создать каталог
curl -s -X POST "$API_URL/api/db/custom_dbtables" \
  -H "Authorization: Bearer $TOKEN" \
  -H "Content-Type: application/json" \
  -d '{"form": {"name": "Quick Notes", "dbname": "quicknotes"}}'

# Добавить поле
curl -s -X POST "$API_URL/api/db/custom_dbfields" \
  -H "Authorization: Bearer $TOKEN" \
  -d 'form[name]=Текст&form[dbname]=content&form[type]=textarea&form[scheme]=custom_quicknotes&submit=1'
```

### Типы полей

| `type` | Описание |
|--------|----------|
| `textbox` | Однострочный текст |
| `textarea` | Многострочный текст |
| `select` | Выпадающий список |
| `checkbox` | Чекбокс |
| `datetime` | Дата и время |
| `photo` | Загрузка файла |
| `select_from_table` | Выбор из другого каталога |

Значения для `select`: передавать в `f_arr` через `\n` (перенос строки).

**Важно**: в JS-коде поля кастомного каталога доступны с префиксом `custom_` — например `note.custom_content`, `note.custom_priority`.

---

## Права доступа (access_db) — ОБЯЗАТЕЛЬНО

### Что такое access_db и чем оно НЕ является

**`access_db` — это не аутентификация.** Пользователь залогинен в Korfix, у него есть сессия или Bearer-токен, `App.fetch('/db/...')` работает — это **аутентификация**.

**`access_db` — это row-level видимость каталога для каждой роли.** Даже если пользователь залогинен и физически может сделать запрос — платформа отфильтрует записи (или вообще вернёт пустоту) в зависимости от его `account_type` и настроек `access_db` для этого каталога.

| Механизм | Что контролирует | Где настраивается |
|---|---|---|
| **Сессия / Bearer** | Может ли запрос вообще быть выполнен (401 если нет) | Логин пользователя или `/db/api` токен |
| **`permissions` в config.json** | Что миниап может делать через `App.fetch` (sandbox) | `config.json` → `permissions.catalogs` |
| **`access_db`** | **Какие записи пользователь с данной ролью может видеть** в каталоге (0/1/2 — нет/все/свои) | Каталог `/db/access_db`, одна запись на (каталог × тенант) |

Без корректной записи в `access_db` миниап **получит пустой список** (или HTTP 200 с `data: []`), даже если API-запрос формально прошёл и у пользователя есть сессия. Это частая причина «у меня всё по документации, но список пустой».

### Результат авто-создания

После создания нового кастомного каталога через `custom_dbtables` платформа автоматически создаёт запись в `access_db` — но **только для администраторов** (`acctype_root=1, acctype_adm=1`), остальные роли = `0` (нет доступа).

Это означает:
- Менеджер, оператор, клиент, бухгалтер и т.д. **не увидят** твой каталог, даже если залогинены и позвали `App.fetch('/db/custom_my_catalog.json')`
- Ответ вернётся со `status: ok`, но `data: []` — это выглядит как «каталог пустой», хотя данные есть
- Если миниап встроен как виджет каталога, который установил админ — работать будет только у админа
- Для корректной работы — обязательно прописать права нужным ролям (см. паттерн `configureAccess` ниже)

### Структура access_db

| Поле | Значение |
|------|----------|
| `dbmodule` | Alias каталога **с** префиксом `custom_` (`custom_my_catalog`) |
| `name` | Название (обычно совпадает с `dbmodule`) |
| `from_group` | Тенант пользователя (из `getUser().group`) |
| `acctype_root` | Права для роли «Администратор» (0/1/2) |
| `acctype_adm` | Права для роли «Менеджер» (0/1/2) |
| `acctype_res`, `acctype_fin`, `acctype_ag1`..`ag6`, `acctype_ec1`..`ec5`, `acctype_b2b1`..`b2b3`, `acctype_md1`..`md3` | Права для соответствующих ролей |

**Значения `acctype_*`:**

| Значение | Доступ | Когда использовать |
|----------|--------|--------------------|
| `0` | Нет доступа (каталог скрыт) | Роль не должна видеть этот каталог вообще |
| `1` | Все записи организации (своего `from_group`) | Совместная работа: задачи, клиенты, сделки — видят все сотрудники |
| `2` | Только свои (`from_auth = свой user_id`) | Персональные данные: заметки, черновики, настройки |

Список ролей **специфичен для инстанса** — на `panel.korfix.ru` один набор, на self-hosted может быть другой. Получить актуальный список: `App.fetch('/db/access_db/sheme.json')` → смотреть поля с префиксом `acctype_` в ответе.

### Best practice: дефолт «2 всем ролям» (self-access)

**Самый безопасный и типовой дефолт — выдать всем ролям значение `2` (только свои записи).** Этого достаточно для большинства приложений: каждый пользователь видит только то, что создал сам.

Когда выбирать другое:
- **Значение `1`** (все записи организации) — для коллаборативных каталогов: задачи (все видят все задачи компании), клиенты CRM, сделки, склад
- **Значение `0`** (нет доступа) — для технических/внутренних каталогов, которые не должен видеть конкретный тип аккаунта
- **Смешанно** — если роли реально отличаются (админ видит всё, менеджер видит всё своей группы, клиент видит только свои заявки)

### Готовый паттерн: «self всем» (рекомендуемый дефолт)

```js
async function configureAccess(catalog, defaultValue = 2) {
    // 1. Получить список полей acctype_* из схемы
    const schema = await App.fetch('/db/access_db/sheme.json');
    const acctypeFields = Object.keys(schema.data || {})
        .filter(k => k.startsWith('acctype_'));

    // 2. Найти auto-created запись для нашего каталога
    const resp = await App.fetch(
        `/db/access_db.json?form[dbmodule]=${catalog}`
    );
    const access = resp.data?.[0];
    if (!access) {
        console.warn(`access_db entry for ${catalog} not found`);
        return;
    }

    // 3. Собрать body со всеми acctype_* = defaultValue
    const body = {
        'form[id]': access.id,
        'form[alias]': access.alias,
        'form[dbmodule]': access.dbmodule,
        submit: 1,
    };
    for (const field of acctypeFields) {
        body[`form[${field}]`] = defaultValue;
    }

    // 4. Обновить запись
    await App.fetch(`/db/access_db/${access.alias}?edit&ajax=1`, {
        method: 'POST',
        body
    });
}

// Использование в installer:
await configureAccess('custom_quicknotes');           // все роли → self (2)
await configureAccess('custom_shared_tasks', 1);      // все роли → all (1)
```

Функция сама подтянет актуальный список ролей текущего инстанса — не надо хардкодить `acctype_adm`, `acctype_b2b2` и т.д.

### Что делать миниапу при установке

Три варианта по убыванию предпочтения:

**1. Автоматически: self всем ролям** (best default — большинство апп-случаев)

```js
await configureAccess('custom_my_catalog');  // см. функцию выше
```

**2. Автоматически: конкретные значения для конкретных ролей** (если нужна коллаборация)

```js
await configureAccess('custom_tasks', 1);  // все видят все
// или точечно:
await App.fetch(`/db/access_db/${access.alias}?edit&ajax=1`, {
    method: 'POST',
    body: {
        'form[id]': access.id,
        'form[alias]': access.alias,
        'form[dbmodule]': access.dbmodule,
        'form[acctype_adm]': 1,      // менеджер — все
        'form[acctype_b2b2]': 2,     // оператор — только свои
        'form[acctype_b2b3]': 0,     // клиент — скрыто
        submit: 1
    }
});
```

**3. Делегировать админу** (если бизнес-логика прав сложная)

В `about` → «Настройка» написать: «После установки откройте `/db/access_db`, найдите запись для `custom_my_catalog`, проставьте права ролям». Это OK если администратор однозначно понимает что выставлять.

### Важные нюансы

- `dbmodule` в `access_db` содержит **полный alias с префиксом** `custom_` — НЕ `dbname` из `custom_dbtables` (там без префикса).
- Изменение `access_db` требует прав уровня **администратор** — обычный пользователь не сможет обновить права через свой токен. Миниап должен либо запускаться в контексте админа при установке, либо делегировать это пользователю.
- `from_group` должен совпадать с тенантом пользователя, иначе запись относится к чужой группе.

---

**Дальше:** [styling.md](styling.md) · **← [Home](index.md)**
