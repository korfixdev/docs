# Фреймы: стандарты и соглашения

> **См. также:** [config-json.md](config-json.md) · [self-provisioning.md](self-provisioning.md) · [dashboards.md](dashboards.md) · [js-api.md](js-api.md)
> **← [Home](index.md)**

Каждый ключ в `config.json → urls` — это фрейм (iframe). Имя фрейма определяет его роль в архитектуре приложения. Следуй этим соглашениям — они проверяются валидатором.

---

## Стандартные типы фреймов

### `install` — экран установки

**Когда использовать:** приложение требует self-provisioning (создаёт custom-каталоги/поля).

**Назначение:** wizard первичной настройки, запускается один раз.

**Обязательные требования:**

- Проверяет существование всех нужных каталогов через `custom_dbtables` (не через `/db/{catalog}.json`)
- **Всегда проверяет РЕАЛЬНЫЙ статус ответа API** — см. раздел «Проверка ответов» ниже
- Сохраняет лог каждого шага в `App.storage` под ключом `install.log`
- При повторном открытии (каталог уже создан): показывает сохранённый лог + кнопки «Переустановить» / «Закрыть»
- Если приложение имеет фрейм `widget` — после успешной установки **авто-добавляет виджет на первый дашборд**
- После завершения установки — перенаправляет в `main` фрейм:
  ```js
  App.navigate(`/db/installed_apps/${params.data.token}?frame=main`);
  ```

**В config.json:**
```json
{
    "urls": { "install": "install.html" },
    "urlsConf": { "install": { "method": "get" } }
}
```

---

### `main` — основной интерфейс

**Когда использовать:** приложение имеет пункт в меню CRM или открывается кнопкой «Открыть».

**Назначение:** первичный экран пользователя.

**Обязательные требования:**

- При загрузке проверяет готовность приложения (каталоги существуют, данные доступны)
- Если приложение не установлено — **перенаправляет на `install` фрейм**:
  ```js
  const App = new VMCRMUserApp();
  const params = await App.getRequestParams();

  const isInstalled = await checkCatalogExists('custom_my_catalog');
  if (!isInstalled) {
      App.navigate(`/db/installed_apps/${params.data.token}?frame=install`);
      return;
  }
  // Приложение готово — рендерим основной UI
  ```
- `urls.main` **обязателен** если приложение добавляет пункт в меню или должно открываться из маркетплейса

**В config.json:**
```json
{
    "urls": { "main": "main.html", "install": "install.html" },
    "urlsConf": { "main": { "method": "get" } },
    "menu": { "tt_tasks": { "frame": "main", "name": "Мой модуль" } }
}
```

---

### `footer` — встроенный виджет каталога

**Когда использовать:** приложение встраивается внутри страниц каталога (под таблицей, над/под карточкой).

**Назначение:** дополнительный UI-блок внутри существующих страниц CRM — не отдельная страница.

**Обязательные требования:**

- Компактная вёрстка: без собственного хедера/навбара
- `App.setFrameSize()` обязателен после каждого render/resize
- Получает контекст текущего каталога через `params.data.catalog`

**В config.json:**
```json
{
    "catalogs": {
        "ag_clients": {
            "catalog.items.footer": { "name": "Мой виджет", "frame": "footer", "width": 6 }
        }
    }
}
```

Пустой ключ каталога `""` применяет виджет ко всем каталогам платформы.
Один html-файл может обслуживать и `footer`, и `widget` — они визуально похожи.

---

### `widget` — компактный виджет дашборда

**Когда использовать:** приложение предоставляет краткую выжимку для рабочего стола.

**Назначение:** встраивание в дашборд через тип `app-frame`.

**Обязательные требования:**

- Компактный размер: ширина ≤ 6 колонок, высота минимальная
- Быстрая загрузка: только суть, без тяжёлых библиотек
- `App.setFrameSize()` обязателен
- Работает при `data.catalog === 'dashboard_widgets'` — стандартный контекст дашборда
- Если приложение имеет self-provisioning (`urls.install`): **install-фрейм обязан авто-установить виджет на первый дашборд**

**В config.json:**
```json
{
    "urls": { "widget": "widget.html" },
    "urlsConf": { "widget": { "method": "get" } },
    "permissions": {
        "catalogs": { "dashboard_widgets": ["read", "write"] }
    }
}
```

---

## Паттерн: авто-установка виджета при install

Вызывается в конце `runInstall()` — после создания каталогов и прав доступа.
Ошибка установки виджета **не прерывает** основную установку — пишем в лог и продолжаем.

```js
async function installWidgetOnDashboard(appToken, widgetName) {
    try {
        const resp = await App.fetch('/api/db/dashboards?limit=999');
        const boards = (resp.data || []).sort((a, b) => (a.prior || 0) - (b.prior || 0));
        if (!boards.length) {
            logLine('⚠ Дашбордов не найдено — виджет можно добавить вручную');
            return;
        }
        const board = boards[0];
        const result = await App.fetch('/db/dashboard_widgets/add?edit&ajax=1', {
            method: 'POST',
            body: {
                'form[alias]': uid(),
                'form[name]': widgetName,
                'form[type]': 'app-frame',
                'form[width]': 6,
                'form[board_id]': board.id,
                'form[options]': JSON.stringify({ app_frame: appToken + ':widget' }),
                'form[from_auth]': currentUserId,
                'form[from_group]': currentUserId,
                submit: 1
            }
        });
        if (!result || result.status === 'error' || result.status === 'no') {
            throw new Error(result?.message || JSON.stringify(result));
        }
        logLine('✓ Виджет добавлен на дашборд «' + board.name + '»');
    } catch (e) {
        logLine('⚠ Не удалось добавить виджет автоматически: ' + e.message);
    }
}

// Как получить appToken и вызвать:
const params = await App.getRequestParams();
const appToken = params.data.token;   // installed_apps.alias текущей установки
await installWidgetOnDashboard(appToken, 'Название виджета');
```

> Примечание: `/api/db/dashboards` (не `/db/`) — возвращает полный список без серверных фильтров. Подробнее: [dashboards.md](dashboards.md).

---

## Проверка ответов API: всегда обязательна

`App.fetch()` **не бросает исключений** при ошибках API — сервер возвращает HTTP 200 с `status: 'error'` или `status: 'no'` в теле. Это частая причина «установка прошла, но ничего не создалось».

**Всегда проверяй реальный статус:**

```js
const resp = await App.fetch('/db/custom_dbtables/add?edit&ajax=1', { method: 'POST', body });

// НЕПРАВИЛЬНО — молча проглатывает ошибку:
// install завершится "успешно", но таблица не создалась
logLine('✓ Таблица создана');  // ← ложный успех

// ПРАВИЛЬНО:
if (!resp || resp.status === 'error' || resp.status === 'no') {
    throw new Error(`Ошибка создания таблицы: ${resp?.message || JSON.stringify(resp)}`);
}
logLine(`✓ Таблица создана (alias: ${resp.alias})`);
```

Применимо ко всем мутирующим запросам: создание каталогов, полей, виджетов, записей.
Для чтения (GET-запросы) — проверять через `resp.data` / `Array.isArray(resp.data)`.

---

## Типовая структура файлов

```
my-app/
├── config.json      # обязателен
├── main.html        # если есть пункт меню / кнопка «Открыть»
├── install.html     # если есть self-provisioning (custom-каталоги)
├── widget.html      # если встраивается на дашборд
├── footer.html      # если встраивается внутрь каталогов (опционально отдельный)
├── logo.svg
└── README.md        # обязателен перед деплоем
```

Один файл может покрывать несколько ролей (например один `widget.html` как `footer` и как `widget`) — допустимо, если UI одинаковый. Соглашение по именам помогает ориентироваться.

---

## Правила валидатора

### Critical (FAIL)

- `main.html` присутствует и есть `urls.install` → в `main.html` должен быть `checkCatalogExists` или `App.navigate(...)` с `frame=install`
- `install.html` содержит мутирующие `App.fetch` без проверки `resp.status`

### Must (WARN)

- `urls.widget` объявлен + есть `urls.install` → в `install.html` должен быть вызов `installWidgetOnDashboard` или аналог
- `urls.widget` объявлен → `permissions.catalogs` должен содержать `dashboard_widgets: ["read", "write"]`

---

**Дальше:** [self-provisioning.md](self-provisioning.md) · **← [Home](index.md)**
