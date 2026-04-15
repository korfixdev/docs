# Self-provisioning: создание структур данных

> **См. также:** [data-api.md](data-api.md) · [catalog-rules.md](catalog-rules.md) · [catalog-settings.md](catalog-settings.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [Home](index.md)**

Приложение может создавать себе каталоги (таблицы) и кастомные поля при установке или первом запуске.

---

## Требования к токену / правам

Для self-provisioning из миниапа (`App.fetch` через сессию) — ничего дополнительного не нужно, сессия пользователя покрывает.

Для работы через Bearer-токен (скрипты, CI, внешние агенты) у токена должны быть классы:

| Класс | Зачем |
|---|---|
| `db_custom_dbtables_get` | Проверка существования каталога в реестре |
| `db_custom_dbtables_post` | Создание кастомного каталога |
| `db_custom_dbfields_post` | Создание полей каталога |
| `db_access_db_get` | Чтение auto-created записи прав (для последующего обновления) |
| `db_access_db_post` | Обновление прав для ролей (`configureAccess` helper) |

Если какого-то класса нет — self-provisioning упадёт с 403 на первом же запросе. Агент-разработчик должен запустить [`korfix-token-audit`](../../devkit-skills/korfix-token-audit/) перед началом работы, чтобы не споткнуться об это посреди процесса.

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

> **Обязательное поле `scheme`** при создании `custom_dbtables`.
> Это template-схема — набор базовых полей (`id`, `alias`, `name`, `ts`, `hidden`, `from_auth`, `from_group`), на основе которой создаётся физическая таблица в БД. Без этого поля платформа не знает из чего строить таблицу и отклоняет запрос.
>
> Сейчас доступно **одно значение**: `coredb_def_catalog` — дефолтная схема каталога. Платформа может расширить варианты в будущем, актуальный список — через `App.fetch('/db/custom_dbtables/sheme.json')` → поле `scheme.arr`.
>
> **Не путать** с полем `scheme` в `custom_dbfields` — там это ссылка на целевой каталог (`custom_X`), для которого создаётся поле.

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
            'form[scheme]': 'coredb_def_catalog',  // ОБЯЗАТЕЛЬНО: template-схема каталога
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

#### Полный паттерн: HTML с экраном установки (ready-made UX)

Правильный экран установки должен:

- Сохранять **лог установки** в `App.storage` — чтобы при повторном открытии пользователь видел что было сделано
- При повторном открытии (каталог уже создан) давать возможность **«Переустановить»** или **«Закрыть»** — а не просто показывать пустоту
- Обрабатывать ошибки и показывать кнопку «Повторить»

```html
<!-- Экран установки (скрыт по умолчанию) -->
<div id="installScreen" style="display:none;">
    <h2 id="installTitle">Установка приложения</h2>
    <p id="installIntro">Для работы нужен каталог данных. Нажмите кнопку установки.</p>
    <div class="install-actions">
        <button id="btnInstall" class="btn btn-primary">Установить структуру данных</button>
        <button id="btnReinstall" class="btn btn-secondary" style="display:none;">Переустановить</button>
        <button id="btnClose" class="btn btn-stroke" style="display:none;">Закрыть</button>
    </div>
    <pre id="installLog" style="background:#f5f5f5;padding:12px;margin-top:16px;max-height:300px;overflow:auto;font-size:12px;white-space:pre-wrap;"></pre>
</div>

<!-- Рабочий UI (скрыт по умолчанию) -->
<div id="mainUI" style="display:none;">
    <!-- ... основной интерфейс приложения ... -->
</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

const LOG_KEY = 'install.log';  // ключ в App.storage для сохранения лога

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
    return rec?.value ?? '';  // ← правильное чтение через .value
}

// ---- init ----

async function init() {
    const exists = await checkCatalogExists();

    if (!exists) {
        // Первый запуск — чистый экран установки, только кнопка Установить
        document.getElementById('installScreen').style.display = '';
        return;
    }

    // Каталог уже есть → показать mainUI, но дать доступ к «переустановить»
    document.getElementById('mainUI').style.display = '';
    loadData();

    // Кнопка «⚙ Переустановить» где-то в настройках → открывает installScreen с логом
    // Здесь можно добавить обработчик на иконку шестерёнки
}

async function openInstallScreen(isReinstall = false) {
    document.getElementById('mainUI').style.display = 'none';
    document.getElementById('installScreen').style.display = '';

    // Показать сохранённый лог из предыдущей установки
    const savedLog = await loadSavedLog();
    document.getElementById('installLog').textContent = savedLog;

    if (isReinstall) {
        document.getElementById('installTitle').textContent = 'Переустановка приложения';
        document.getElementById('installIntro').textContent = 'Структура уже создана. Можно переустановить (пересоздаст поля) или закрыть.';
        document.getElementById('btnInstall').style.display = 'none';
        document.getElementById('btnReinstall').style.display = '';
        document.getElementById('btnClose').style.display = '';
    }
}

// ---- button handlers ----

document.getElementById('btnInstall').addEventListener('click', async () => {
    const btn = document.getElementById('btnInstall');
    btn.disabled = true;
    btn.textContent = 'Установка...';
    logLine('Начинаем установку...');

    try {
        await runInstall((msg) => logLine(msg));  // runInstall вызывает logLine для каждого шага
        logLine('✓ Установка завершена');
        await saveLog();

        // Переход к основному UI
        document.getElementById('installScreen').style.display = 'none';
        document.getElementById('mainUI').style.display = '';
        loadData();
    } catch (e) {
        logLine(`✗ Ошибка: ${e.message}`);
        await saveLog();
        btn.disabled = false;
        btn.textContent = 'Повторить установку';
    }
});

document.getElementById('btnReinstall').addEventListener('click', async () => {
    if (!confirm('Переустановить? Это пересоздаст недостающие поля. Данные в существующих записях сохранятся.')) return;
    const btn = document.getElementById('btnReinstall');
    btn.disabled = true;
    btn.textContent = 'Переустановка...';
    logLine('--- Переустановка ---');

    try {
        await runInstall((msg) => logLine(msg));
        logLine('✓ Переустановка завершена');
        await saveLog();
        alert('Готово');
        document.getElementById('installScreen').style.display = 'none';
        document.getElementById('mainUI').style.display = '';
    } catch (e) {
        logLine(`✗ Ошибка: ${e.message}`);
        await saveLog();
        btn.disabled = false;
        btn.textContent = 'Повторить переустановку';
    }
});

document.getElementById('btnClose').addEventListener('click', () => {
    document.getElementById('installScreen').style.display = 'none';
    document.getElementById('mainUI').style.display = '';
});

init();
</script>
```

**Ключевые моменты:**

1. `LOG_KEY = 'install.log'` — лог сохраняется в `App.storage` (изолированное хранилище миниапа), переживает рестарт браузера
2. При повторном открытии installScreen — подтягивается сохранённый лог через `loadSavedLog()` (чтение через `.value`, не напрямую)
3. Две кнопки: **«Установить»** для первого запуска и **«Переустановить»** для повторного запуска с существующим каталогом
4. Кнопка **«Закрыть»** — чтобы просто выйти из installScreen без действий
5. `runInstall(cb)` принимает callback для логирования каждого шага (`logLine('Создаю поле X...')` изнутри)
6. Лог сохраняется даже при ошибке — можно потом посмотреть что было

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

Когда создаётся запись в `custom_dbtables` (через UI, API/Bearer или `App.fetch` из миниапа) — срабатывает afteradd-хук, который **автоматически создаёт запись в `access_db`** с правами **только для администраторов**: `acctype_root=1, acctype_adm=1`, остальные роли = `0` (нет доступа).

Это означает:

- Каталог **сразу виден** пользователю с ролью root/adm (обычно создателю приложения — администратору)
- **Не виден** другим ролям (менеджер, оператор, клиент и т.д.) — они получат пустой список `data: []` при `App.fetch('/db/custom_X.json')`
- Если миниап предназначен для других ролей — приложение **должно обновить** `access_db` с нужными `acctype_*` значениями (см. паттерн `configureAccess` ниже)

Одна физическая таблица `crm__custom_{dbname}` шарится между аккаунтами (разными `from_group`), но `access_db` создаётся per-аккаунт — поэтому каждый аккаунт независимо настраивает права для своих ролей.

На `access_db` действует уникальный ключ `UNIQUE (dbmodule, from_auth, from_group)`. Операционно для одного каталога у одного аккаунта — **одна запись** (с `from_auth = ID владельца каталога`, `from_group = ID аккаунта`). Права внутри записи заданы колонками `acctype_*` — не нужно (и не следует) создавать дополнительные записи чтобы "расширить доступ".

!!! warning "Анти-паттерн: `from_auth = 0` в access_db"

    В обычных каталогах `from_auth = 0` означает "запись принадлежит всей группе" (видна всем участникам `from_group`), и это легитимный паттерн. На уровне save-логики сервер разрешает этот `0` сознательно.

    **В `access_db` так делать нельзя.** Видимость каталога роли определяется значениями `acctype_*` (`0`/`1`/`2`) — сам механизм row-ownership через `from_auth` для этой таблицы **не применяется платформой**. Создавать вторую запись с `(dbmodule, 0, from_group)` "чтобы было видно всем" — бессмысленно, это только добавит дубликат в списке `/db/access_db`, а реальных прав не расширит.

    Правильно: одна запись `(dbmodule, owner_id, from_group)` с нужными `acctype_*`.

**Важно — серверная подстановка from_auth/from_group:**

С апреля 2026 платформа (при включённом `FEATURES_USED.auth_role`) автоматически подставляет `from_group` и `from_auth` из сессии/токена при INSERT/UPDATE в любой каталог. Клиенту больше **не нужно** передавать их вручную в `form[from_auth]`/`form[from_group]` — сервер возьмёт значения из авторизации. Если клиент всё же передаёт, эти значения проходят валидацию:

- non-admin не может указать чужой `from_group` / чужой `from_auth` — сервер заменит на свой
- admin может точечно назначить владельца (`from_auth = конкретный ID`) или перенести в другую группу
- `from_auth = 0` всегда разрешён (общая для группы) — для `access_db` это анти-паттерн (см. выше)

Это убирает старую боль "записи создаются с `from_group=0` если не передать руками". Теперь ручная передача — только для явного трансфера (админ).

### Когда приложение должно думать об access_db

При проектировании миниапа **обязательно определиться** какие роли должны видеть каталог и как:

- **Только админы** — оставить как есть (дефолт). Подходит для служебных/настроечных каталогов.
- **Персональные данные всех ролей** — каждая роль → `acctype_* = 2` (self, только свои записи). Подходит для заметок, ToDo, настроек, черновиков.
- **Коллаборация** — выбранные роли → `acctype_* = 1` (все записи организации). Подходит для задач, клиентов, сделок.
- **Смешанно** — точечно по ролям (админ видит всё, клиент только свои, аналитик только read-only — через дополнительные механизмы прав).

Если из контекста задачи **не ясно** какую схему выбрать — **спроси пользователя**:
> «Этот каталог — для какой роли? Видно только админу, или нужно чтобы разные роли видели свои записи, или чтобы все видели все?»

Не угадывай. От этого зависит правильная настройка `access_db`.

### Обновление прав через API

Паттерн `configureAccess` — обновляет (или создаёт если нет) запись в access_db всем ролям текущего инстанса одним значением:

```js
async function configureAccess(catalog, defaultValue = 2) {
    const schema = await App.fetch('/db/access_db/sheme.json');
    const acctypeFields = Object.keys(schema.data || {})
        .filter(k => k.startsWith('acctype_'));

    // Проверить существует ли запись
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

    if (!existing) {
        // Создаём новую запись
        body['form[alias]'] = Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
        const resp = await App.fetch('/db/access_db/add?edit&ajax=1', {
            method: 'POST',
            body
        });
        if (!resp || resp.status === 'error' || resp.status === 'no') {
            throw new Error(`Failed to create access_db entry: ${JSON.stringify(resp)}`);
        }
    } else {
        // Обновляем существующую
        body['form[id]'] = existing.id;
        body['form[alias]'] = existing.alias;
        const resp = await App.fetch(`/db/access_db/${existing.alias}?edit&ajax=1`, {
            method: 'POST',
            body
        });
        if (!resp || resp.status === 'error' || resp.status === 'no') {
            throw new Error(`Failed to update access_db entry: ${JSON.stringify(resp)}`);
        }
    }
}
```

Раньше пример показывал только update-ветку с `if (!access) return` — это **ошибка**, если запись access_db не была создана автоматически (API-путь), миниап молча не получил бы никаких прав. Теперь правильный паттерн всегда — **create-or-update**.

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

Функция работает как **create-or-update** — если запись access_db отсутствует (типичный кейс при self-provisioning через API, когда afteradd-хук не отработал), она создаётся. Если уже есть — обновляется.

```js
function uid() {
    return Date.now().toString(36) + Math.random().toString(36).substr(2, 8);
}

async function configureAccess(catalog, defaultValue = 2) {
    // 1. Получить список полей acctype_* из схемы
    const schema = await App.fetch('/db/access_db/sheme.json');
    const acctypeFields = Object.keys(schema.data || {})
        .filter(k => k.startsWith('acctype_'));

    // 2. Проверить существует ли запись для нашего каталога
    const existing = (await App.fetch(
        `/db/access_db.json?form[dbmodule]=${catalog}`
    )).data?.[0];

    // 3. Собрать body со всеми acctype_* = defaultValue
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
        // Создаём новую запись — платформа через API не создаёт автоматически
        body['form[alias]'] = uid();
        resp = await App.fetch('/db/access_db/add?edit&ajax=1', {
            method: 'POST',
            body
        });
    } else {
        // Обновляем существующую
        body['form[id]'] = existing.id;
        body['form[alias]'] = existing.alias;
        resp = await App.fetch(`/db/access_db/${existing.alias}?edit&ajax=1`, {
            method: 'POST',
            body
        });
    }

    // 4. Проверить результат — App.fetch не бросает при status:'error' / 'no'
    if (!resp || resp.status === 'error' || resp.status === 'no') {
        throw new Error(
            `configureAccess(${catalog}) failed: ${resp?.message || JSON.stringify(resp)}`
        );
    }
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

- **Запись `access_db` создаётся автоматически** после INSERT в `custom_dbtables` с дефолтом `acctype_root=1, acctype_adm=1` (остальные роли = 0). Это работает одинаково через UI и через API.
- `dbmodule` в `access_db` содержит **полный alias с префиксом** `custom_` — НЕ `dbname` из `custom_dbtables` (там без префикса).
- Изменение `access_db` требует прав уровня **администратор** — обычный пользователь не сможет обновить права через свой токен. Миниап должен либо запускаться в контексте админа при установке, либо делегировать это пользователю.
- `from_group` должен совпадать с тенантом пользователя, иначе запись относится к чужой группе.
- **При удалении каталога** (soft-delete `hidden=1`) запись в `access_db` **не удаляется автоматически** — платформа сохраняет права на случай восстановления каталога или если таблица шарится с другими аккаунтами. Очистка вручную при необходимости.
- **Физическая таблица** `crm__custom_{dbname}` **также не удаляется** при soft-delete каталога — в ней могут быть данные других аккаунтов (разные `from_group` в одной общей таблице). Удаление только через SQL/админа.

---

**Дальше:** [styling.md](styling.md) · **← [Home](index.md)**
