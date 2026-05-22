# Quick Start — vibe-coding on Korfix in 10 minutes

> **See also:** [miniapps/getting-started.md](miniapps/getting-started.md) · [miniapps/rules.md](miniapps/rules.md) · [plugin-update.md](plugin-update.md)
> **← [Home](index.md)**

This is the landing page for people who heard about Korfix and vibe-coding and want to try it themselves. By the end you'll have a registered account, an installed plugin, a working API token, and a clear idea of what to ask Claude to build you an app.

Korfix is a platform where your data lives in catalogs, your tools live as miniapps on top of them, and AI has direct access to all of it via MCP. Vibe-coding here means you describe the app you want in plain language, and AI assembles it end-to-end — deploys the database, builds the UI, publishes to your marketplace. No DevOps, no servers, no UI tutorials.

## Step 1. Register — 1 minute

Open [vibe.korfix.info](https://vibe.korfix.info), enter your email, confirm. No card required, no trial period — the platform is free for users.

After signing in you land **not on an empty dashboard**. Your account already has a set of pre-installed miniapps — Roadmap, Kanban, Finance, Prompt Library, n8n Monitor, Games Hub and others. The dashboard with widgets is in place too. All of this was assembled the same way you're about to assemble your own — by describing it to AI. It's your first answer to the question "what can you actually build on this platform?".

Click around the menu, open a couple of apps, take a look at the catalogs. Then move on to building your own.

## Step 2. Install the korfix-devkit plugin

`korfix-devkit` is a Claude Code plugin that turns Claude into a miniapp developer: it knows the platform's formats, designs solutions, writes code, validates, and deploys.

Install commands are on the [home page](index.md): add the `korfixdev/marketplace` marketplace, install `korfix-devkit`, reload plugins. See [Update plugins](plugin-update.md) for upgrades.

The plugin ships with agents (`korfix-analyst`, `korfix-miniapp-dev`, `korfix-miniapp-validator`), skills, and the platform documentation bundled.

**Not using Claude Code?** If you're in Cursor, Codex or another AI — just point the model at [docs.korfix.info](https://docs.korfix.info). Quality is lower (no agents and no automatic validation), but it works.

## Step 3. Create an API token

The token is what lets AI publish apps to your marketplace and work with your data.

In the platform: **Settings → API** (or type `api` into the menu search on the left — fastest way to find the section). Click **Add**.

In the form, fill in:

- **Purpose** — a label for yourself, e.g. "vibe coding".
- **Catalogs and methods** — the token's permissions.

### What rights to give the token

| Catalog | Why it's needed | Level |
|---------|-----------------|-------|
| `marketplace` | publishing and updating miniapps via API | **required** (read + write) |
| `custom_dbtables` | letting your miniapps create their own catalogs from the installer | **recommended** (read + write) |
| `custom_dbfields` | letting your miniapps add fields to their own catalogs | **recommended** (read + write) |
| `installed_apps` | working with the registry of installed apps | optional |
| Business catalogs (`ag_*`, `tt_*`, `b2b_*`, `md_*`, …) | only if AI needs to work with specific business data | skip for personal apps |

**Minimum to deploy your own apps** — `marketplace` with write permissions. Without it, the agent cannot upload the zip.

**Minimum for self-provisioning** (when an app creates its own catalogs at install time) — add `custom_dbtables` and `custom_dbfields` with write permissions. Almost any useful miniapp with its own data needs this.

If unsure — tick "all permissions" and narrow it down later once you know what you actually need. You can revoke or recreate the token at any time.

Once saved, the token lives in your panel — there's no need to copy it elsewhere or store it in files. It just needs to exist and be active.

## Step 4. Hand the token to Claude — without copy-paste

Don't put the token into env variables, configs, scripts, or chats. Just launch Claude Code and ask it to publish something. The `korfix-miniapp-dev` agent will ask which instance and which token to use, and won't deploy without confirmed values.

You enter it once — and the plugin works with that token for the rest of the session.

## Step 5. First prompt

You're ready. Tell Claude what you want, in your own words:

> I need an app for Korfix: expense tracking. I create wallets and expense categories, log transactions. Dashboard widget — a pie chart by category and a list of recent transactions.

What happens:

1. **`korfix-analyst`** asks clarifying questions — about transfers between wallets, currency, default time range for the chart. You answer in words.
2. **`korfix-miniapp-dev`** assembles the app: `config.json`, the installer with the right catalogs and relations, the dashboard widget. Packages it as a zip.
3. **`korfix-miniapp-validator`** reviews the code against the release checklist (fresh context, independent run).
4. After you confirm — deploy to your marketplace using the token from Step 4.

A new app appears in your marketplace. You install it — the installer rolls out the "Wallets", "Expense categories" and "Transactions" catalogs with the right schema, the dashboard widget shows up. Open and use.

Need changes? Same conversation: "add a month filter", "drop the recent transactions list, keep only the chart". AI revises and republishes.

## Bonus. AI reads data directly via MCP

This is a separate story, perpendicular to vibe-coding. If you want **Claude Desktop, Cursor or n8n** to see your catalogs directly and answer questions like "how much did I spend on food" — use the **MCP URL** from Step 3.

In `claude_desktop_config.json`:

```json
{
  "mcpServers": {
    "korfix": {
      "type": "sse",
      "url": "PASTE_MCP_URL_FROM_SETTINGS"
    }
  }
}
```

Restart Claude Desktop — it sees your catalogs. Building apps via `korfix-devkit` and reading data via MCP are two independent scenarios; you can use both at once.

## Where to go next

| Where | Why |
|-------|-----|
| [miniapps/rules.md](miniapps/rules.md) | miniapp sandbox rules: what's allowed, what isn't |
| [miniapps/getting-started.md](miniapps/getting-started.md) | how a miniapp works under the hood — if you want to edit by hand |
| [miniapps/self-provisioning.md](miniapps/self-provisioning.md) | how an app creates its own catalogs at install time |
| [miniapps/deploy.md](miniapps/deploy.md) | deployment details via API |
| [gamedev/index.md](gamedev/index.md) | if you want to write games rather than business apps |

---

**Next:** [miniapps/rules.md](miniapps/rules.md) · **← [Home](index.md)**
