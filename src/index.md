# Korfix Documentation

Public documentation for the Korfix platform — ERP, miniapp marketplace, AI development SDK.

## Sections

- 📘 [miniapps/](miniapps/index.md) — marketplace miniapp development (HTML+JS+CSS in iframe, platform API)

Planned: `catalogs/` (business catalog reference), `workflows/` (automation recipes), `backend/` (public framework architecture).

## Using these docs

**Miniapp developers** — start with [miniapps/rules.md](miniapps/rules.md) and [miniapps/getting-started.md](miniapps/getting-started.md).

**AI agents** — via Claude Code plugins:

**Step 1 — Add Korfix marketplace** (once):
```
korfixdev/marketplace
```
In Claude Code: `/plugin` → **Add marketplace** → paste the line above.

**Step 2 — Install the plugin you need:**

For miniapp development:
```
/plugin install korfix-devkit@korfixdev
```
For business data queries:
```
/plugin install korfix-assistant@korfixdev
```

**Step 3 — Activate:**
```
/reload-plugins
```

Each plugin ships with agents, skills, and documentation.

## Related

- [korfixdev/devkit](https://github.com/korfixdev/devkit) — Claude Code plugin for miniapp development
- [korfixdev/assistant](https://github.com/korfixdev/assistant) — Claude Code plugin for business queries
- [korfix.ru](https://korfix.ru) — main product
- [panel.korfix.ru](https://panel.korfix.ru) — production instance

## Contributing

Issues and PRs welcome: [github.com/korfixdev/docs](https://github.com/korfixdev/docs).

## License

CC BY-SA 4.0.

## Contact

info@korfix.ru

---

# Документация Korfix

Публичная документация платформы Korfix — ERP, маркетплейс миниапов, SDK для AI-разработки.

## Разделы

- 📘 [miniapps/](miniapps/index.md) — разработка маркетплейс-миниапов (HTML+JS+CSS в iframe, работа с API)

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
