# Adapt a Telegram Mini App

An app written for Telegram (using `telegram-web-app.js` and `window.Telegram.WebApp`) runs in Korfix
with **minimal changes**.

> **← [Overview](index.md)** · [SDK reference & auth](sdk.md)

---

## 1. Swap the SDK script

Replace the Telegram script with the Korfix **compatibility adapter**:

```html
<!-- before -->
<!-- <script src="https://telegram.org/js/telegram-web-app.js"></script> -->

<!-- after -->
<script src="https://YOUR-KORFIX-HOST/templates/def/chat/korfix-web-app-tg-compat.js"></script>
```

The adapter pulls in the Korfix SDK automatically (from the same host it was loaded from) and exposes
`window.Telegram.WebApp` as an alias of `window.KorfixApp`. **You include just this one script** — no
need to add `korfix-web-app.js` separately.

!!! tip "Running on your own domain"
    The adapter loads the SDK relative to its own URL, so loading the adapter from the Korfix host works
    even when your app is served from a different domain. You can also vendor both files
    (`korfix-web-app.js` + `korfix-web-app-tg-compat.js`) side by side with your app.

After this, your existing calls keep working:

```js
const tg = window.Telegram.WebApp;
tg.ready();
tg.expand();
const user = tg.initDataUnsafe.user;
tg.MainButton.setText('Confirm').show().onClick(() => tg.sendData(JSON.stringify(payload)));
```

---

## 2. Point your backend at the Korfix secret

The `initData` signing scheme is **identical** to Telegram
(`secret_key = HMAC_SHA256(key="WebAppData", message=secret)`), so your validation code does **not**
change — only the **secret value** does:

- In Telegram the secret is your **bot token**.
- In Korfix the secret is the **per‑app secret** you set when registering the app in the room
  (ℹ️ info → **Apps** tab).

So either set the app's secret in Korfix to match what your backend already expects, or update the
secret constant in your backend to the one issued in Korfix. See the validation snippets in
[SDK reference & auth](sdk.md#validate-initdata-on-your-backend).

---

## 3. Know the differences

**`initData` fields.** Korfix sends a subset: `user`, `chat_instance`, `chat_type`, `auth_date`, `hash`.
Telegram‑specific fields (`query_id`, `chat`, `start_param`, `signature`, `receiver`) are **not** sent —
apps that depend on them need a fallback.

**API coverage.** The adapter maps the common surface:

| Supported | Not supported (yet) |
|-----------|---------------------|
| `ready` / `expand` / `close` | `CloudStorage` |
| `sendData` | `requestContact` / `requestWriteAccess` |
| `MainButton` / `BackButton` | `openLink` / `openTelegramLink` |
| `showPopup` / `showAlert` / `showConfirm` | invoices / payments |
| `HapticFeedback` | QR scanner, geolocation, biometrics |
| `onEvent` / `offEvent`, `themeParams` | |

Calls to unsupported methods are no‑ops or `undefined`; guard for them if your app relies on them.

---

## Migration checklist

- [ ] Replaced `telegram-web-app.js` with `korfix-web-app-tg-compat.js` from the Korfix host
- [ ] Backend validates with the Korfix per‑app secret (same scheme as before)
- [ ] App handles missing `query_id` / `start_param` gracefully
- [ ] No hard dependency on unsupported Bot API methods
- [ ] App registered in the room with that secret
