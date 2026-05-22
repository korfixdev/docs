# Using the docs without the Claude Code plugin

> **← [Home](index.md)**

The `korfix-devkit` plugin is built for **Claude Code** and relies on its skills/agents mechanism.
If you work in **Codex**, **Cursor**, **Gemini CLI**, or any other AI tool without plugin/agent/skill
support — this page explains how to get the same result.

---

## How the plugin works (for context)

In Claude Code the plugin provides:

- **Skills** — markdown files with instructions that the AI reads on demand
- **Agents** — specialised roles with rule sets
- **Docs** — API and platform reference

Without that mechanism, you simply **pass the skill and doc contents into the model's context manually**.

---

## Quick start for Codex / Cursor / other AI

### 1. Main entry point

Before any miniapp work, give the AI the contents of these files:

```
docs/miniapps/rules.md           — sandbox rules (required)
docs/miniapps/getting-started.md — first miniapp
```

### 2. Pick the docs that match your task

| Task | Files to pass into context |
|------|----------------------------|
| Create a new miniapp | `rules.md` + `getting-started.md` + `config-json.md` + `styling.md` |
| Work with catalog data | `data-api.md` + `js-api.md` |
| Persist app settings | `storage-and-hooks.md` |
| Create a catalog at install time | `self-provisioning.md` + `data-api.md` |
| Dashboard widget | `dashboards.md` + `config-json.md` |
| Deploy | `deploy.md` |
| Pre-release checks | `checklist.md` |

### 3. Skills — pass `SKILL.md` contents directly

Each skill in the plugin is a file at `skills/<name>/SKILL.md`. Paste it straight into the context:

| What you need | File |
|---------------|------|
| Catalog CRUD | `skills/korfix-crud-data/SKILL.md` |
| JS inside the iframe | `skills/korfix-js-api/SKILL.md` |
| Catalog schema | `skills/korfix-catalog-schema/SKILL.md` |
| Miniapp config | `skills/korfix-miniapp-config/SKILL.md` |
| Pre-deploy validation | `skills/korfix-miniapp-validate/SKILL.md` |
| Self-provisioning | `skills/korfix-self-provisioning/SKILL.md` |
| Token audit | `skills/korfix-token-audit/SKILL.md` |

### 4. Agents — pass `agents/*.md` contents directly

Agent roles live in `agents/`:

| Role | File |
|------|------|
| Miniapp developer | `agents/korfix-miniapp-dev.md` |
| Requirements analyst | `agents/korfix-analyst.md` |
| Architect | `agents/korfix-architect.md` |
| Gamedev | `agents/korfix-gamedev.md` |
| Validator | `agents/korfix-miniapp-validator.md` |

---

## Sample prompt for Codex

```
You are a miniapp developer for the Korfix platform.
Working rules: [paste contents of agents/korfix-miniapp-dev.md]
Sandbox rules: [paste contents of docs/miniapps/rules.md]
JS API: [paste contents of skills/korfix-js-api/SKILL.md]

Task: build a miniapp that tracks customer applications.
```

---

## Notes

- All documentation paths are relative to the platform inside the iframe (`/db/catalog.json`) — no domain needed.
- Environment variables (`KORFIX_API_URL`, `KORFIX_TOKEN`) — set them via `.env` or directly in the AI's instructions.
- Up-to-date online docs: [docs.korfix.info](https://docs.korfix.info).

---

**← [Home](index.md)**
