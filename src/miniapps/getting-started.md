# Первое приложение за 15 минут

> **См. также:** [rules.md](rules.md) · [config-json.md](config-json.md) · [js-api.md](js-api.md) · [deploy.md](deploy.md)
> **← [Home](index.md)**

End-to-end: от пустой папки до работающего виджета в CRM.
Перед началом обязательно прочитай [rules.md](rules.md).

---

## Что такое миниап

HTML-страница в `<iframe>`, встроенная в CRM-платформу. Общается с платформой через `postMessage` (JS-класс `VMCRMUserApp`). Может читать и писать данные каталогов через `App.fetch()`.

**В zip — только** HTML, JS, CSS, картинки, шрифты, md. Серверный код (PHP) **запрещён**. Если нужна серверная логика — используй удалённое приложение (`install_url` вместо zip).

---

## Шаг 1. Структура

```
my-app/
├── config.json      # обязательно
├── widget.html      # фрейм (один или несколько)
├── logo.svg         # опционально
```

## Шаг 2. Минимальный `config.json`

```json
{
    "name": "Моё приложение",
    "version": "1.0.0",
    "description": "Считает элементы в каталоге",
    "about": "## Что делает\nПоказывает количество записей в каталоге.\n\n## Где появляется в CRM\n- Пункт меню «Счётчик» после каталога Задачи\n- Виджет под списком любого каталога\n\n## Как пользоваться\nОткройте любой каталог — увидите счётчик под таблицей.",
    "logo": "logo.svg",
    "urls": {
        "widget": "widget.html"
    },
    "urlsConf": {
        "widget": { "method": "get" }
    },
    "permissions": {
        "catalogs": { "*": ["read"] }
    },
    "menu": {
        "tt_tasks": { "frame": "widget", "name": "Счётчик" }
    },
    "catalogs": {
        "": {
            "catalog.items.footer": {
                "name": "Счётчик записей",
                "frame": "widget"
            }
        }
    }
}
```

**Важное про `menu`:**
- **Ключ** = alias каталога в боковом меню, **ПОСЛЕ которого** появится новый пункт. Например `"tt_tasks"` вставит пункт сразу за разделом «Задачи».
- **Значение** — объект с `frame` (имя из `urls`) и `name` (подпись в меню).
- Если хочешь несколько пунктов меню — укажи несколько ключей: `"tt_tasks": {...}, "ag_clients": {...}`.

**Про `catalogs."" `:**
- Пустой ключ `""` — настройка применяется **ко всем каталогам** платформы. Удобно для универсальных виджетов.
- Конкретный ключ (`"tt_tasks"`) — настройка только для этого каталога.

Полная спека всех полей, точек встраивания и permissions → [config-json.md](config-json.md).

## Шаг 3. Минимальный `widget.html`

```html
<!DOCTYPE html>
<html lang="ru">
<head>
<meta charset="UTF-8">
<style>
    body { font-family: 'Open Sans', -apple-system, sans-serif; margin: 0; padding: 12px; overflow: hidden; }
    .count { font-size: 20px; font-weight: 600; color: var(--primary-color, #2e7d32); }
</style>
</head>
<body>
<div id="app">Загрузка…</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

const params = await App.getRequestParams();
const { catalog } = params.data;

const resp = await App.fetch(`/db/${catalog}.json?page=1`);
const total = resp.total ?? resp.data?.length ?? 0;

document.getElementById('app').innerHTML =
    `<div>В каталоге <b>${catalog}</b>: <span class="count">${total}</span> записей</div>`;

requestAnimationFrame(() => App.setFrameSize(null, document.body.scrollHeight));
</script>
</body>
</html>
```

Что здесь важно:
- `VMCRMUserApp` грузится с портала, а не с CDN — [js-api.md](js-api.md)
- `App.fetch()` использует сессию пользователя, токен не нужен — [data-api.md](data-api.md)
- `App.setFrameSize()` подгоняет высоту iframe — подробнее про авторесайз и стили в [styling.md](styling.md)

### ⚠️ `/db/` vs `/api/db/` — не перепутай

Код внутри миниапа (в iframe) использует **`/db/{catalog}.json`** через `App.fetch`. Это работает по **сессии** пользователя (cookie), токен не нужен.

Если хочешь тестировать тот же запрос снаружи (через curl, скрипт, Postman) — нужно использовать **`/api/db/{catalog}`** с Bearer-токеном, не `/db/`:

```bash
# Внутри iframe — через App.fetch:
await App.fetch('/db/tt_tasks.json')                       # ← работает

# Снаружи (curl, тесты, автоматизация):
curl "https://panel.korfix.ru/api/db/tt_tasks?limit=10" \
  -H "Authorization: Bearer YOUR_TOKEN"                    # ← работает

# НЕ ДЕЛАЙ так снаружи — получишь 302 на логин:
curl https://panel.korfix.ru/db/tt_tasks.json              # ✗ 302
```

Подробнее: [data-api.md](data-api.md) — раздел «Ключевое правило: какой endpoint откуда».

## Шаг 4. Упаковать и задеплоить

```bash
cd my-app
zip -r /tmp/my-app.zip config.json widget.html logo.svg

curl -X POST "https://panel-korfix.vnn.ru/api/db/marketplace/{ID}?token={TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

Получение ID приложения, CI/CD-скрипт, обновление версии → [deploy.md](deploy.md).

## Шаг 5. Установить и проверить

1. Открой маркетплейс: `/db/marketplace`
2. Найди своё приложение в списке → нажми **Установить**
3. После установки открой любой каталог — под таблицей появится счётчик

> Список установленных приложений доступен в `/db/installed_apps` (заполняется автоматически), но установка запускается из маркетплейса, не оттуда.

---

## Что дальше

| Нужно | Файл |
|-------|------|
| Передавать данные в каталог | [data-api.md](data-api.md) |
| Добавить реакцию на сохранение записи | [catalog-rules.md](catalog-rules.md) или [storage-and-hooks.md](storage-and-hooks.md) |
| Сделать виджет дашборда | [dashboards.md](dashboards.md) |
| Создать свои каталоги при установке | [self-provisioning.md](self-provisioning.md) |
| Понять какие каталоги вообще доступны | [korfix-catalogs.md](korfix-catalogs.md) |
| Оформить в стиле платформы | [styling.md](styling.md) |
| Перед релизом | [checklist.md](checklist.md) |

---

**Дальше:** [config-json.md](config-json.md) · **← [Home](index.md)**
