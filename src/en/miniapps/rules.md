# Miniapp Development Rules (required reading)

> **← [Home](index.md)** · This is the first document. Read before you start.

A miniapp is an HTML/JS/CSS application running in an isolated iframe inside Korfix CRM. These rules cover security, correctness, and portability.

---

## Security

**Keep tokens and secrets in env, not in code.**

- `KORFIX_TOKEN` — in developer environment variables. Never commit to the miniapp's git, never include in the zip, never expose in the UI.
- If a token ends up in a commit — revoke it immediately in `/db/api` and issue a new one.
- Use **different tokens** for production and staging.
- Tokens should have **minimum required permissions**: only the catalogs the miniapp actually needs, only the necessary methods (read/write).
- Secrets in `config.json` (external API keys etc.) — store in `App.storage`, not in application files.

## API access — use the right channels

**Inside the iframe:** use `App.fetch('/db/...')`.
**Outside (webhooks, n8n, scripts):** `Authorization: Bearer <token>` with `/api/...`.

- **Never** call `window.fetch()` directly from the miniapp — CORS will block it.
- **Never** use browser cookies/sessions to bypass `App.fetch` — the iframe is isolated, these mechanisms are unavailable and can give incorrect results.
- **Never** hardcode the instance URL (`panel.korfix.ru` etc.) — use relative paths so the app works on any instance.

## What can go in the zip

**Allowed:** `.html`, `.js`, `.css`, `.svg`, `.png`, `.jpg`, `.webp`, `.ico`, `.gif`, `.json`, `.md`, `.woff`, `.woff2`, `.ttf`, `.eot`.

**Prohibited:** any executable/server-side extensions — `.php`, `.exe`, `.sh`, `.py`, `.rb`, etc. The platform will reject a zip containing such files.

**Server logic** — only via a "remote app" (`install_url` instead of `urls` in `config.json`). Server code then lives on your server, and the miniapp receives data from the platform via `afterSave` webhooks.

## Portability

- **`urls` in config.json — relative paths** (`"main": "index.html"`) for zip-packaged apps. The platform fills in the path.
- **Don't reference a specific domain** in miniapp code or `config.json` for local apps. The miniapp must work on `panel.korfix.ru`, `acme.korfix.ru`, and any other instance without changes.
- **Don't hardcode article IDs, catalog IDs, group IDs** — fetch them via `App.fetch('/db/{catalog}/sheme.json')` or `catalog_schema()`.

## Record ownership

When creating any record via the API:

- **`alias`** — generate explicitly: `Date.now().toString(36) + Math.random().toString(36).substr(2, 8)`. Don't rely on auto-generation, especially for bulk inserts.
- **`from_auth`** — ID of the creating user. Fetch via `catalog_schema().from_auth.arr` — take the key that's not `0`.
- **`from_group`** — tenant (group). Usually equals `from_auth` for personal records or the group ID for shared ones.

Without `from_auth`/`from_group` the record is created under the super-admin and won't be visible to regular users. This is a bug, not a feature.

## Principle: don't look for workarounds

If something doesn't work via `App.fetch` or the documented APIs — **don't search for a workaround**. Stop, ask a question: either the documentation is incomplete, or this action genuinely shouldn't be done.

Examples of what **not to do**:
- Embedding a `<script>` from an external domain to bypass CORS (instead — `App.fetch` via `postMessage`)
- Calling `parent.postMessage` directly with a hardcoded target origin (use `App` methods — they know where they came from)
- Storing state in the iframe's `localStorage` (isolated; use `App.storage` instead)
- Using `<iframe src="/db/...">` to bypass the API (the platform UI isn't guaranteed to be embeddable)

If the API is missing something — open an issue at [korfixdev/docs](https://github.com/korfixdev/docs) or ask in the community. Don't write code that depends on undocumented platform behavior.

---

## See also

- [getting-started.md](getting-started.md) — first app from scratch
- [data-api.md](data-api.md) — request formats and filters
- [js-api.md](js-api.md) — VMCRMUserApp: methods and events
- [deploy.md](deploy.md) — packaging and deploy
- [checklist.md](checklist.md) — pre-release checklist

**Next:** [getting-started.md](getting-started.md) · **← [Home](index.md)**
