# Marketplace — Разработка миниапов

Документация для разработки маркетплейс-приложений (миниапов) платформы Korfix / VMCRM.

> **Первый документ:** [rules.md](rules.md) — правила песочницы, запреты, принципы. **Читать до начала работы.**
> Серверная архитектура, хуки ядра, инфраструктура — в [../backend/INDEX.md](../backend/INDEX.md)

---

## Быстрый старт

1. [rules.md](rules.md) — что можно и что нельзя (обязательно)
2. [getting-started.md](getting-started.md) — первое приложение за 15 минут
3. [deploy.md](deploy.md) — упаковка и загрузка через API
4. [checklist.md](checklist.md) — проверка перед релизом

---

## По группам

### Core API — как общаться с платформой
| Файл | Описание |
|------|----------|
| [config-json.md](config-json.md) | Точки встраивания, permissions, дашборд-виджеты |
| [js-api.md](js-api.md) | `VMCRMUserApp`: методы, события, CORS, навигация |
| [data-api.md](data-api.md) | CRUD каталогов, `form[]` vs plain, нормализация, фильтры |
| [storage-and-hooks.md](storage-and-hooks.md) | `App.storage` (KV), вебхуки, `afterSave` |

### UI и стилизация
| Файл | Описание |
|------|----------|
| [styling.md](styling.md) | CSS-переменные, компоненты, Chart.js, адаптив, авторесайз |
| [dashboards.md](dashboards.md) | Дашборды и виджеты: типы, параметры, создание через API |

### Справочники каталогов (какие данные можно читать)
| Файл | Описание |
|------|----------|
| [korfix-catalogs.md](korfix-catalogs.md) | Все доступные каталоги Korfix ERP |
| [favorites-menu.md](favorites-menu.md) | `favorites_menu` — избранное и стартовая страница |
| [systempush-settings.md](systempush-settings.md) | `systempush_settings` — push-подписки |
| [account-help.md](account-help.md) | `account_help`, `service_help` — контекстная справка |
| [access-statuses.md](access-statuses.md) | `access_statuses` — права на статусы по ролям |
| [bitrix24-sync.md](bitrix24-sync.md) | `bitrix24_sync` — двусторонняя синхронизация с Bitrix24 |

### Фичи платформы
| Файл | Описание |
|------|----------|
| [self-provisioning.md](self-provisioning.md) | Создание каталогов и полей при установке |
| [catalog-rules.md](catalog-rules.md) | Декларативные правила afterSave: inherit / calc / aggregate / validate |
| [catalog-settings.md](catalog-settings.md) | Настройки отображения каталога (колонки, порядок) |
| [db-views.md](db-views.md) | VIEW-представления: объединение каталогов через LEFT JOIN |

### Lifecycle — от кода до релиза
| Файл | Описание |
|------|----------|
| [rules.md](rules.md) | Правила и запреты |
| [getting-started.md](getting-started.md) | Первое приложение |
| [deploy.md](deploy.md) | Упаковка, загрузка, CI/CD |
| [checklist.md](checklist.md) | Перед релизом |

---

## Задача → какие файлы читать

| Хочу... | Файлы |
|---------|-------|
| Встроить виджет под список каталога | [config-json.md](config-json.md) + [js-api.md](js-api.md) + [styling.md](styling.md) |
| Добавить пункт главного меню | [config-json.md](config-json.md) + [getting-started.md](getting-started.md) |
| Читать/писать данные каталога | [data-api.md](data-api.md) + [js-api.md](js-api.md) |
| Создать свой каталог при установке | [self-provisioning.md](self-provisioning.md) + [data-api.md](data-api.md) |
| Реагировать на сохранение записи | [catalog-rules.md](catalog-rules.md) (декларативно) **или** [storage-and-hooks.md](storage-and-hooks.md) (вебхук) |
| Сделать виджет дашборда | [dashboards.md](dashboards.md) + [config-json.md](config-json.md) |
| Объединить данные двух каталогов | [db-views.md](db-views.md) |
| Работать с push-уведомлениями | [systempush-settings.md](systempush-settings.md) |
| Синхронизироваться с Bitrix24 | [bitrix24-sync.md](bitrix24-sync.md) |
| Отправлять запросы с сервера (вне iframe) | [data-api.md](data-api.md) (раздел Bearer-токены) |
| Хранить настройки приложения | [storage-and-hooks.md](storage-and-hooks.md) |
| Стилизовать UI под платформу | [styling.md](styling.md) |
| Задеплоить приложение | [deploy.md](deploy.md) |
| Проверить приложение перед релизом | [checklist.md](checklist.md) |

---

## Соглашения документации

- В каждом файле вверху блок **«См. также»** — смежные темы.
- В конце — **«Дальше: X»** для линейного чтения и **← INDEX** для возврата.
- Примеры кода — рабочие, копипастой должны запускаться.
- Эталонные приложения: `panel.korfix.ru/vmcrm-apps/` (проверенные) и `panel.korfix.ru/korfix-apps/` (прототипы).
