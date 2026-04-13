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
}
```

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

После создания нового кастомного каталога он **не виден** обычным пользователям. Платформа автоматически создаёт запись в служебном каталоге `access_db` для нового каталога, но **только с правами для администраторов** (`acctype_root=1, acctype_adm=1`), остальные роли — `0` (нет доступа).

Это означает:
- Менеджер, оператор, клиент, бухгалтер и т.д. **не увидят** твой каталог в меню/списках
- Если миниап встроен как виджет каталога, который установил админ — работать будет только у админа
- Для корректной работы — обязательно прописать права нужным ролям

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

| Значение | Доступ |
|----------|--------|
| `0` | Нет доступа (каталог скрыт) |
| `1` | Полный доступ (видит все записи) |
| `2` | Только свои (видит только записи, где `from_auth = свой user_id`) |

Список ролей **специфичен для инстанса** — на `panel.korfix.ru` одни роли, на self-hosted может быть другой набор. Получить актуальный список: `App.fetch('/db/access_db/sheme.json')` → смотреть поля `acctype_*` в ответе.

### Что делать миниапу при установке

Два варианта:

**1. Автоматически обновить access_db из миниапа** (если знаешь какие роли должны иметь доступ):

```js
// Найти запись access_db для своего каталога
const resp = await App.fetch(
    '/db/access_db.json?form[dbmodule]=custom_quicknotes'
);
const access = resp.data?.[0];

if (access) {
    // Обновить права для менеджеров (acctype_adm) и операторов (acctype_b2b2)
    await App.fetch(`/db/access_db/${access.alias}?edit&ajax=1`, {
        method: 'POST',
        body: {
            'form[id]': access.id,
            'form[alias]': access.alias,
            'form[dbmodule]': access.dbmodule,
            'form[acctype_adm]': 1,   // менеджер — все записи
            'form[acctype_b2b2]': 2,  // оператор — только свои
            submit: 1
        }
    });
}
```

**2. Попросить админа после установки**:

В `about` раздел «Настройка» явно написать: «После установки откройте `/db/access_db`, найдите запись для `custom_my_catalog`, проставьте нужные права для ролей».

### Важные нюансы

- `dbmodule` в `access_db` содержит **полный alias с префиксом** `custom_` — НЕ `dbname` из `custom_dbtables` (там без префикса).
- Изменение `access_db` требует прав уровня **администратор** — обычный пользователь не сможет обновить права через свой токен. Миниап должен либо запускаться в контексте админа при установке, либо делегировать это пользователю.
- `from_group` должен совпадать с тенантом пользователя, иначе запись относится к чужой группе.

---

**Дальше:** [styling.md](styling.md) · **← [Home](index.md)**
