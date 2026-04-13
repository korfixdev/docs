# korfixdev/docs

Public documentation for the [Korfix](https://korfix.ru) platform — single source of truth for all consumers:
- Miniapp developers (read at [docs.korfix.info](https://docs.korfix.info))
- AI agents via [korfixdev/devkit](https://github.com/korfixdev/devkit) and [korfixdev/assistant](https://github.com/korfixdev/assistant) plugins (pull from this repo)
- Static site builds (MkDocs Material → GitHub Pages)

## Navigation

See [src/index.md](src/index.md).

## Sections

| Folder | Audience | Content |
|--------|----------|---------|
| `src/miniapps/` | Marketplace miniapp developers | SDK: config.json, JS API, data API, styling, deploy |

Planned: `src/catalogs/` (business catalog reference), `src/workflows/` (automation recipes), `src/backend/` (public framework architecture).

## Contributing

1. Fork → edit → PR. New file — add link to `src/miniapps/index.md` and include "See also" block at top.
2. Every doc has a top block `> **См. также:**` with 2-4 links to related topics and `> **← [Home](index.md)**`. At the bottom — `**Дальше:** X · **← [Home](index.md)**` for linear reading, or just `← Home` for reference docs.
3. Code examples should be working — runnable via copy-paste.

## Site build

The `src/` folder is the content root for MkDocs. On every push to `main`, GitHub Actions builds the Material-themed site and deploys to GitHub Pages → [docs.korfix.info](https://docs.korfix.info).

## Related

- [korfix.ru](https://korfix.ru) — Korfix platform (product)
- [korfixdev/devkit](https://github.com/korfixdev/devkit) — miniapp development plugin
- [korfixdev/assistant](https://github.com/korfixdev/assistant) — business data queries plugin
- [github.com/korfixdev](https://github.com/korfixdev) — organization

## License

CC BY-SA 4.0 — attribution required, derivatives under same license.

## Contact

info@korfix.ru

---

# korfixdev/docs — на русском

Публичная документация платформы [Korfix](https://korfix.ru) — источник правды для всех потребителей:
- Разработчики миниапов (читают на [docs.korfix.info](https://docs.korfix.info))
- AI-агенты через плагины [korfixdev/devkit](https://github.com/korfixdev/devkit) и [korfixdev/assistant](https://github.com/korfixdev/assistant) (подтягивают этот репо)
- Сборка статического сайта (MkDocs Material → GitHub Pages)

## Навигация

См. [src/index.md](src/index.md).

## Разделы

| Папка | Аудитория | Содержание |
|-------|-----------|-----------|
| `src/miniapps/` | Разработчики маркетплейс-миниапов | SDK: config.json, JS API, data API, стилизация, деплой |

В будущем: `src/catalogs/` (справочник бизнес-каталогов), `src/workflows/` (рецепты автоматизации), `src/backend/` (архитектура ядра).

## Контрибьюция

1. Fork → правка → PR. Новый файл — обязательно добавь ссылку в `src/miniapps/index.md` и блок «См. также» в шапку.
2. Каждый документ содержит вверху блок `> **См. также:**` с 2-4 ссылками на смежные темы и `> **← [Home](index.md)**`. Внизу — `**Дальше:** X · **← [Home](index.md)**` для линейного чтения, или только `← Home` для справочников.
3. Примеры кода — рабочие, должны запускаться копипастой.

## Сборка сайта

Папка `src/` — корень контента для MkDocs. При каждом push в `main` GitHub Actions собирает Material-сайт и деплоит на GitHub Pages → [docs.korfix.info](https://docs.korfix.info).

## Связанные

- [korfix.ru](https://korfix.ru) — платформа Korfix (продукт)
- [korfixdev/devkit](https://github.com/korfixdev/devkit) — плагин для разработки миниапов
- [korfixdev/assistant](https://github.com/korfixdev/assistant) — плагин для бизнес-запросов
- [github.com/korfixdev](https://github.com/korfixdev) — организация

## Лицензия

CC BY-SA 4.0 — атрибуция обязательна, производные работы под той же лицензией.

## Контакт

info@korfix.ru
