# Update plugins

Claude Code does not auto-check third-party marketplaces by default. To pull the latest `korfix-devkit` or `korfix-assistant`, use one of the methods below.

> One marketplace (`korfixdev`) ships both plugins — a single `marketplace update` covers them together.

## Manual update (always works)

Fetch fresh metadata from the marketplace:

```
/plugin marketplace update korfixdev
```

Activate the new version in the current session:

```
/reload-plugins
```

Restarting Claude Code has the same effect as `/reload-plugins`.

## Auto-update (opt-in, one-time setup)

Type `/plugin` → **Marketplaces** tab → select `korfixdev` → enable **Auto-update**.

After that Claude Code will check on every session start and prompt you to run `/reload-plugins` when a new version is available.

## Env overrides (rarely needed)

- `DISABLE_AUTOUPDATER=1` — disable all auto-updates (Claude Code itself + plugins).
- `FORCE_AUTOUPDATE_PLUGINS=1` — force plugin auto-updates even if Claude Code's own auto-update is off.

## Check installed version

Open `/plugin` in Claude Code, or inspect the plugin registry:

```
~/.claude/plugins/installed_plugins.json
```

---

# Обновление плагинов

Claude Code **не проверяет** сторонние маркетплейсы автоматически по умолчанию. Чтобы подтянуть свежий `korfix-devkit` или `korfix-assistant`, используй один из способов ниже.

> Оба плагина живут в одном маркетплейсе (`korfixdev`) — одна команда `marketplace update` обновит сразу оба.

## Ручное обновление (всегда работает)

Подтянуть свежие метаданные из маркетплейса:

```
/plugin marketplace update korfixdev
```

Активировать новую версию в текущей сессии:

```
/reload-plugins
```

Перезапуск Claude Code делает то же, что и `/reload-plugins`.

## Автообновление (включается один раз)

Набери `/plugin` → вкладка **Marketplaces** → выбери `korfixdev` → включи **Auto-update**.

После этого Claude Code при старте сессии сам проверит обновления и подскажет запустить `/reload-plugins`, если что-то вышло.

## Env-переменные (редко нужны)

- `DISABLE_AUTOUPDATER=1` — выключить все авто-обновления (и сам Claude Code, и плагины).
- `FORCE_AUTOUPDATE_PLUGINS=1` — форсить авто-обновление плагинов, даже если авто-обновление Claude Code выключено.

## Посмотреть установленную версию

Открой `/plugin` в Claude Code или загляни в реестр плагинов:

```
~/.claude/plugins/installed_plugins.json
```
