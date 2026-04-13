# Деплой приложения

> **См. также:** [rules.md](rules.md) · [getting-started.md](getting-started.md) · [checklist.md](checklist.md) · [config-json.md](config-json.md)
> **← [INDEX](INDEX.md)**

Пошаговое создание, упаковка, загрузка через API и обновление миниапов.

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
- `urlsConf` — метод запроса (get для локальных, post для remote)
- `catalogs` или `menu` — куда встраивается

### Шаг 4: Написать HTML фрейм

Шаблон:

```html
<!DOCTYPE html>
<html lang="ru">
<head>
    <meta charset="UTF-8">
    <style>
        * { margin: 0; padding: 0; box-sizing: border-box; }
        body { font-family: -apple-system, BlinkMacSystemFont, 'Segoe UI', sans-serif; background: transparent; }
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

### Шаг 5: Упаковать в zip

```bash
# В директории приложения:
zip -r ../my-app.zip config.json widget.html app.js style.css
```

**Допустимые расширения в zip**: html, htm, txt, js, png, jpg, jpeg, webp, ico,
gif, css, json, svg, map, md, eot, woff2, ttf, woff.

### Шаг 6: Загрузить в маркетплейс

**Через интерфейс:**
1. `/db/marketplace` -> Добавить
2. Заполнить имя, загрузить zip в поле `doc1`
3. Сохранить

**Через API (рекомендуется для вайбкодинга):**
```bash
zip -r my-app.zip config.json widget.html app.js
curl -X POST "https://panel.korfix.ru/api/db/marketplace/{ID}?token={TOKEN}" \
  -F "doc1=@my-app.zip;type=application/zip"
```

Достаточно **только `doc1`** — zip-файл. Имя, описание, теги подтягиваются
из `config.json` внутри архива. Лишние поля (`form[name]`, `form[id]` и т.д.) не нужны.

### Шаг 7: Установить приложение

1. `/db/installed_apps` -> Добавить
2. Выбрать приложение из списка
3. Сохранить -> фреймы появятся в указанных точках встраивания

---

## Автодеплой через API

Загрузку zip можно автоматизировать через REST API, не заходя в интерфейс.

### Деплой новой версии одной командой

```bash
zip -r my-app.zip config.json widget.html app.js
curl -X POST "https://panel.korfix.ru/api/db/marketplace/{ID}?token={TOKEN}" \
  -F "doc1=@my-app.zip;type=application/zip"
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
APP_ALIAS="820f955291ded191e1008447791cca4e"

cd "$APP_DIR"
zip -r /tmp/app-deploy.zip config.json *.html *.js *.css 2>/dev/null

RESPONSE=$(curl -s -X POST "$API_URL/api/db/marketplace/$APP_ID?token=$TOKEN" \
  -F "doc1=@/tmp/app-deploy.zip;type=application/zip")

echo "$RESPONSE"
```

### Требования к токену

Токен из `/db/api` должен иметь доступ к каталогу `marketplace` (метод POST).
Добавьте `db_marketplace_post` в "Классы API" токена.

### Цикл вайбкодинга

1. Ассистент редактирует html/js/css файлы приложения
2. Пакует в zip
3. Деплоит через curl одной командой
4. Результат виден в браузере сразу после обновления страницы

---

## Обновление приложения

Процесс обновления:
1. Перезалить zip (через форму или API-деплой)
2. В маркетплейсе появляется бейдж "обновить"
3. Нажать кнопку "Обновить" на странице приложения
4. Фреймы приложения начинают раздаваться с новым кодом

При API-деплое шаги 1-3 происходят автоматически.

---

**Дальше:** [checklist.md](checklist.md) · **← [INDEX](INDEX.md)**
