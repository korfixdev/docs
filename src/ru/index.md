# Документация Korfix

Публичная документация платформы Korfix — ERP, маркетплейс миниапов, SDK для AI-разработки.

## Разделы

- 📘 [miniapps/](miniapps/index.md) — разработка маркетплейс-миниапов (HTML+JS+CSS в iframe, работа с API)
- 🎮 [gamedev/](gamedev/index.md) — игры и геймификация (кошелёк Korn, квесты, лидерборды, внутриигровой магазин)

В будущем: `catalogs/` (справочник бизнес-каталогов), `workflows/` (рецепты автоматизации), `backend/` (архитектура ядра).

## Как использовать

**Разработчикам миниапов** — начни с [miniapps/rules.md](miniapps/rules.md) и [miniapps/getting-started.md](miniapps/getting-started.md).

**AI-агентам** — через Claude Code plugins:

**Шаг 1 — Добавь маркетплейс Korfix** (один раз):
```
korfixdev/marketplace
```
В Claude Code: `/plugin` → **Add marketplace** → вставь строку выше.

**Шаг 2 — Установи нужный плагин:**

Для разработки миниапов:
```
/plugin install korfix-devkit@korfixdev
```
Для бизнес-запросов к данным:
```
/plugin install korfix-assistant@korfixdev
```

**Шаг 3 — Активируй:**
```
/reload-plugins
```

**Обновление плагинов:** Claude Code не опрашивает сторонние маркетплейсы автоматически — см. [Обновление плагинов](plugin-update.md) про ручное и авто-обновление.

## Связанные ресурсы

- [korfixdev/devkit](https://github.com/korfixdev/devkit)
- [korfixdev/assistant](https://github.com/korfixdev/assistant)
- [korfix.ru](https://korfix.ru) — основной сайт
- [panel.korfix.ru](https://panel.korfix.ru) — рабочий инстанс

## Контрибьюция

Issues и PR — [github.com/korfixdev/docs](https://github.com/korfixdev/docs).

## Лицензия

CC BY-SA 4.0.

## Контакт

info@korfix.ru
