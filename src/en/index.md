# Korfix Documentation

Public documentation for the Korfix platform — ERP, miniapp marketplace, AI development SDK.

## Sections

- 📘 [miniapps/](miniapps/index.md) — marketplace miniapp development (HTML+JS+CSS in iframe, platform API)
- 🎮 [gamedev/](gamedev/index.md) — games and gamification mechanics (Korn wallet, quests, leaderboards, in-game shop)

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

**Keeping plugins up to date:** Claude Code does not poll third-party marketplaces automatically — see [Update plugins](plugin-update.md) for manual and auto-update options.

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
