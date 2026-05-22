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

See [CONTRIBUTING.md](CONTRIBUTING.md) for the full rules. Two things to remember:

1. **English first.** All content authored under `src/en/`, then mirrored to `src/ru/`. PRs include both files.
2. **No `.md` at `src/` root.** Files live in `src/en/` and `src/ru/` only — root conflicts abort the build.

Page conventions (See also block at top, Next at bottom, runnable code, no secrets) — same in both languages.

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

Полные правила — в [CONTRIBUTING.md](CONTRIBUTING.md). Главное:

1. **Сначала английский.** Контент пишется в `src/en/`, потом зеркалится в `src/ru/`. PR содержит обе версии.
2. **В корне `src/` файлов `.md` нет.** Только `src/en/` и `src/ru/` — дубли в корне ломают сборку.

Page conventions (блок «См. также» наверху, «Дальше» внизу, рабочий код, без секретов) — те же для обоих языков.

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
