# Деплой приложения

> **См. также:** [rules.md](rules.md) · [getting-started.md](getting-started.md) · [checklist.md](checklist.md) · [config-json.md](config-json.md)
> **← [Home](index.md)**

Пошаговое создание, упаковка, загрузка через API и обновление миниапов.

> ⚠ **Важно:** деплой и обновление миниапа всегда идут через каталог `/db/marketplace` (создание/перезалив zip). Каталог `/db/installed_apps` — это **только реестр уже установленных** приложений, заполняется автоматически. Туда писать руками или через API **не нужно**.

---

## Пошаговое создание приложения

### Шаг 1: Определить тип

| Тип | Когда использовать |
|-----|-------------------|
| Виджет в footer каталога | Дополнительная визуализация под списком |
| Виджет в карточке | Доп. информация на странице элемента |
| Пункт меню | Полноэкранное приложение (календарь, канбан) |
| itemsAction | Действие над элементом (попап) |
| afterSave hook | Серверная реакция на сохранение (только remote) |

### Шаг 2: Создать файлы

```bash
mkdir my-app
cd my-app
# Создать config.json и widget.html
```

### Шаг 3: Написать config.json

Определить:

- `urls` — фреймы приложения
- `urlsConf` — метод запроса (`get` для локальных, `post` для remote)
- `catalogs` или `menu` — куда встраивается
- `permissions` — какие каталоги читает/пишет

Полная спека и пример → [config-json.md](config-json.md).

### Шаг 4: Написать HTML фрейм

Шаблон:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: 'Open Sans', -apple-system, sans-serif; background: transparent; }
    </style>
</head>
<body>
<div id="app">Loading...</div>

<script type="module">
import VMCRMUserApp from '/templates/def/db/marketplace/vmcrm-user-app.js';
const App = new VMCRMUserApp();

async function init() {
    const params = await App.getRequestParams();
    const { catalog, itemId, token } = params.data;

    // Загрузить данные
    const resp = await App.fetch(`/db/${catalog}.json`);

    // Отрисовать UI
    document.getElementById('app').innerHTML = `
        <p>Каталог: ${catalog}, элементов: ${resp.data?.length || 0}</p>
    `;

    // Подогнать размер фрейма
    App.setFrameSize(null, document.body.scrollHeight + 10);
}

init();
</script>
</body>
</html>
```

### Шаг 5: Проверить по чеклисту

Перед упаковкой пройди по [checklist.md](checklist.md). Особое внимание:

- `config.json` валиден, поле `about` со всеми 5 разделами
- Все файлы из `urls` существуют
- `App.setFrameSize()` вызывается после рендера
- `App.fetch()` вместо `window.fetch()`
- `font-size ≥ 16px` в input/select/textarea (iOS Safari)

### Шаг 6: Упаковать в zip

```bash
# В директории приложения:
zip -r /tmp/my-app.zip config.json widget.html *.js *.css *.svg
```

**Допустимые расширения в zip**: html, htm, txt, js, png, jpg, jpeg, webp, ico, gif, css, json, svg, map, md, eot, woff2, ttf, woff.

**Запрещённые**: php, exe, sh — платформа отклонит zip.

### Шаг 7: Загрузить в маркетплейс

Загрузка приложения = создание/обновление записи в каталоге `/db/marketplace`.

#### Через интерфейс

1. Открой `/db/marketplace`
2. Нажми **Добавить** (для нового) или открой существующее (для обновления)
3. Загрузи zip-файл в поле `doc1`
4. Сохрани

#### Через API (рекомендуется для вайбкодинга)

```bash
curl -X POST "https://panel.korfix.ru/api/db/marketplace/{ID}?token={TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

Достаточно **только `doc1`** — zip-файл. Имя, описание, теги подтягиваются из `config.json` внутри архива. Лишние поля (`form[name]`, `form[id]` и т.д.) не нужны.

### Шаг 8: Установить приложение

Установка = пользовательское действие в UI маркетплейса.

1. Открой маркетплейс: `/db/marketplace`
2. Найди своё приложение в списке (по `name` из `config.json`)
3. Нажми кнопку **Установить** на карточке приложения
4. Фреймы появятся в указанных в `config.json` точках встраивания

> Реестр `installed_apps` заполняется автоматически — туда писать руками не нужно.

---

## Автодеплой через API

Загрузку zip можно автоматизировать через REST API, не заходя в интерфейс.

### Деплой новой версии одной командой

```bash
zip -r /tmp/my-app.zip config.json widget.html *.js *.css

curl -X POST "https://panel.korfix.ru/api/db/marketplace/{ID}?token={TOKEN}" \
  -F "doc1=@/tmp/my-app.zip;type=application/zip"
```

### Получение ID и alias приложения

```bash
curl -H "Authorization: Bearer {TOKEN}" \
  "https://panel.korfix.ru/api/db/marketplace?filter[name]=My App"
```

### Проверка версии после деплоя

```bash
curl -H "Authorization: Bearer {TOKEN}" \
  "https://panel.korfix.ru/api/db/marketplace/{ID}"
# В поле appconfig.version — версия из config.json
```

### CI/CD скрипт

```bash
#!/bin/bash
APP_DIR="./my-app"
API_URL="https://panel.korfix.ru"
TOKEN="your-api-token"
APP_ID="50"

cd "$APP_DIR"
zip -r /tmp/app-deploy.zip config.json *.html *.js *.css 2>/dev/null

RESPONSE=$(curl -s -X POST "$API_URL/api/db/marketplace/$APP_ID?token=$TOKEN" \
  -F "doc1=@/tmp/app-deploy.zip;type=application/zip")

echo "$RESPONSE"
```

### Требования к токену

Токен из `/db/api` должен иметь доступ к каталогу `marketplace` (метод POST). Добавь `db_marketplace_post` в **«Классы API»** токена.

### Цикл вайбкодинга

1. Ассистент редактирует html/js/css файлы приложения
2. Проверяет на соответствие [checklist.md](checklist.md)
3. Пакует в zip
4. Деплоит через `curl` одной командой в `/db/marketplace`
5. Результат виден в браузере сразу после обновления страницы

---

## Обновление приложения

Процесс обновления:

1. Ассистент редактирует html/js/css файлы приложения
2. Проверяет на соответствие [checklist.md](checklist.md)
3. Пакует в zip
4. Перезаливает zip (через форму или API-деплой) в `/db/marketplace`
5. В маркетплейсе появляется бейдж «обновить»
6. Пользователь нажимает кнопку **Обновить** на странице приложения
7. Фреймы приложения начинают раздаваться с новым кодом

> При API-деплое шаги 1-4 идут автоматически. Шаги 5-7 — действия пользователя в интерфейсе маркетплейса.

---

**Дальше:** [checklist.md](checklist.md) · **← [Home](index.md)**
