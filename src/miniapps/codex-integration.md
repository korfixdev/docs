# Использование документации без Claude Code плагина

> **← [Home](index.md)**

Плагин `korfix-devkit` написан под **Claude Code** и использует механизмы skills/agents.
Если вы работаете в **Codex**, **Cursor**, **Gemini CLI** или другом AI-инструменте без поддержки
plugin/agent/skill — эта страница объясняет как получить тот же результат.

---

## Как работает плагин (для понимания)

В Claude Code плагин даёт:
- **Skills** — markdown-файлы с инструкциями, которые AI читает по запросу
- **Agents** — специализированные роли с набором правил
- **Docs** — справочная документация по API и платформе

Без этого механизма нужно просто **передавать контент skills и docs в контекст вручную**.

---

## Быстрый старт для Codex / Cursor / другого AI

### 1. Главная точка входа

Перед началом любой работы с миниапами передай AI содержимое этих файлов:

```
docs/miniapps/rules.md          — правила песочницы (обязательно)
docs/miniapps/getting-started.md — первый миниап
```

### 2. Выбери нужные документы по задаче

| Задача | Файлы для передачи в контекст |
|--------|-------------------------------|
| Создать новый миниап | `rules.md` + `getting-started.md` + `config-json.md` + `styling.md` |
| Работа с данными каталогов | `data-api.md` + `js-api.md` |
| Хранение настроек приложения | `storage-and-hooks.md` |
| Создание каталога при установке | `self-provisioning.md` + `data-api.md` |
| Дашборд-виджет | `dashboards.md` + `config-json.md` |
| Деплой | `deploy.md` |
| Проверка перед релизом | `checklist.md` |

### 3. Вместо skill — передай содержимое SKILL.md

Каждый skill в плагине — это файл `skills/<name>/SKILL.md`. Передай его напрямую в контекст:

| Нужная функция | Файл |
|----------------|------|
| Работа с каталогами (CRUD) | `skills/korfix-crud-data/SKILL.md` |
| JS внутри iframe | `skills/korfix-js-api/SKILL.md` |
| Схема каталога | `skills/korfix-catalog-schema/SKILL.md` |
| Конфиг миниапа | `skills/korfix-miniapp-config/SKILL.md` |
| Проверка перед деплоем | `skills/korfix-miniapp-validate/SKILL.md` |
| Self-provisioning | `skills/korfix-self-provisioning/SKILL.md` |
| Аудит токена | `skills/korfix-token-audit/SKILL.md` |

### 4. Вместо agent — передай содержимое agents/*.md

Роли агентов описаны в `agents/`:

| Роль | Файл |
|------|------|
| Разработчик миниапов | `agents/korfix-miniapp-dev.md` |
| Аналитик требований | `agents/korfix-analyst.md` |
| Архитектор | `agents/korfix-architect.md` |
| Gamedev | `agents/korfix-gamedev.md` |
| Валидатор | `agents/korfix-miniapp-validator.md` |

---

## Пример промпта для Codex

```
Ты разработчик миниапов для платформы Korfix.
Правила работы: [вставь содержимое agents/korfix-miniapp-dev.md]
Правила песочницы: [вставь содержимое docs/miniapps/rules.md]
JS API: [вставь содержимое skills/korfix-js-api/SKILL.md]

Задача: создай миниап для учёта заявок клиентов.
```

---

## Замечания

- Все пути в документации — относительные внутри iframe (`/db/catalog.json`), не нужно домен
- Переменные окружения (`KORFIX_API_URL`, `KORFIX_TOKEN`) задай через .env или напрямую в инструкции AI
- Актуальная онлайн-версия документации: [docs.korfix.info](https://docs.korfix.info)

---

**← [Home](index.md)**
