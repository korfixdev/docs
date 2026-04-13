# Self-provisioning: создание структур данных

> **См. также:** [data-api.md](data-api.md) · [catalog-rules.md](catalog-rules.md) · [catalog-settings.md](catalog-settings.md) · [korfix-catalogs.md](korfix-catalogs.md)
> **← [INDEX](INDEX.md)**

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

**Дальше:** [styling.md](styling.md) · **← [INDEX](INDEX.md)**
