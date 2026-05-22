# Contributing to korfix-docs

Public documentation for the [Korfix](https://korfix.ru) platform.
The site is built by MkDocs Material + `mkdocs-static-i18n` and deployed to
[docs.korfix.info](https://docs.korfix.info) on every push to `main`.

## Authoring rule: English first

**All content lives under `src/en/`. The Russian tree under `src/ru/` is a translation, not a parallel source.**

This is the load-bearing convention. Edit in the wrong order and the site goes out of sync.

### Workflow

1. **Write in English.** Add or edit the page under `src/en/...`.
2. **Mirror to Russian.** Update the matching file under `src/ru/...` so the two trees stay in parity.
3. **Add to navigation if it's a new page.** In `mkdocs.yml` → `nav:` register the path once (the i18n plugin resolves it per language). Add the page title to `nav_translations:` in the `i18n` plugin block so the Russian site shows the Russian title.
4. **Submit one PR with both files.** Don't merge an EN change without the RU counterpart or you ship a broken locale.

### Why EN-first

- The default locale is English (`default: true` for `en` in `mkdocs.yml`). `docs.korfix.info/` serves the English site; `/ru/` is the alternate. Missing locales fall back to the default — so an EN-only page works, an RU-only page is invisible to the EN audience.
- The reader base is wider in English: AI plugins (`korfix-devkit`, `korfix-assistant`) pull documentation for LLM agents, and English is the lingua franca there.
- Russian content older than its English counterpart was the failure mode that produced inconsistent translations historically. Reversing the order fixes it at the source.

### What about root `src/`?

Don't put `.md` files at `src/` root. The `mkdocs-static-i18n` plugin with `docs_structure: folder` reads only `src/en/` and `src/ru/`; files at the root conflict with the default locale and abort the build. The two folders are the only authoring surface.

## Page conventions

Every page (both EN and RU) keeps the same structure:

- **Top block:** `> **See also:** [page1](page1.md) · [page2](page2.md)` with 2–4 links to related topics, followed by `> **← [Home](index.md)**` (or the localised equivalent in RU).
- **Bottom block:** `**Next:** [next-topic](next-topic.md) · **← [Home](index.md)**` for linear reading, or just `**← [Home](index.md)**` for reference docs.
- **Code blocks are runnable.** Copy-paste should execute. If a snippet requires a value the reader provides, mark it clearly (`YOUR_TOKEN`, `PASTE_MCP_URL_FROM_SETTINGS`).
- **No secrets.** Never paste real tokens, MCP URLs, account IDs. Placeholders only.

## Adding a new page

1. Create `src/en/<section>/<slug>.md` with the page content.
2. Create `src/ru/<section>/<slug>.md` with the Russian translation.
3. Update `mkdocs.yml`:
   - Add the page to `nav:` under the right section.
   - Add the page title to `nav_translations:` (`English title: Russian title`).
4. Link the new page from neighbouring pages' **See also** blocks.

## Site build

`src/` is the MkDocs content root. On every push to `main`, GitHub Actions runs `mkdocs build` and deploys to GitHub Pages → [docs.korfix.info](https://docs.korfix.info). CI fails on:

- A plugin in `mkdocs.yml` that isn't in `requirements.txt`.
- A file present at `src/` root with the same name as one in `src/en/` (locale conflict).
- A nav entry pointing to a missing file in the default locale.

If CI is red, the live site is stuck on the previous successful build.

## Related

- [korfix.ru](https://korfix.ru) — Korfix platform (product)
- [korfixdev/devkit](https://github.com/korfixdev/devkit) — Claude Code plugin for miniapp development
- [korfixdev/assistant](https://github.com/korfixdev/assistant) — Claude Code plugin for business data queries

## License

CC BY-SA 4.0 — attribution required, derivatives under the same license.

---

# Контрибьюция в korfix-docs (резюме на русском)

Главное правило: **сначала EN, потом RU**.

1. Правишь файл под `src/en/...`.
2. Переводишь / зеркалишь в `src/ru/...`.
3. Если страница новая — добавляешь её в `nav:` и `nav_translations:` в `mkdocs.yml`.
4. PR содержит обе версии разом.

EN — дефолтный язык сайта (`docs.korfix.info/`), RU — `/ru/`. Отсутствующие переводы откатываются к EN; обратное — нет. AI-плагины подтягивают доку именно из EN-дерева.

В корень `src/` `.md`-файлы не клади — там только `src/en/` и `src/ru/`. `mkdocs-static-i18n` падает на конфликте root и default-локали.

Соглашения о структуре страницы (блок **See also** наверху, **Next** внизу, нет секретов в примерах) — те же для обоих языков.
