# Build your first Messenger App

A Messenger App is any web page that includes the Korfix Web App SDK and is opened inside a chat room.

> **← [Overview](index.md)** · next: [SDK reference & auth](sdk.md)

---

## 1. Include the SDK

```html
<script src="https://YOUR-KORFIX-HOST/templates/def/chat/korfix-web-app.js"></script>
```

`YOUR-KORFIX-HOST` is the host where the messenger runs (for example `panel.korfix.ru` or
`vibe.korfix.app`). The SDK is a tiny, dependency‑free file. If your app must run on several Korfix
instances, you can also **vendor** (ship a copy of) `korfix-web-app.js` with your app — it has no
host‑specific code.

The SDK exposes a global object: **`window.KorfixApp`**.

---

## 2. Signal readiness and read the user

```html
<script src="https://YOUR-KORFIX-HOST/templates/def/chat/korfix-web-app.js"></script>
<script>
  KorfixApp.ready();          // tell the host the app is ready
  KorfixApp.expand();         // (optional) request more height

  const user = KorfixApp.initDataUnsafe.user;   // { id, first_name, username, ... }
  document.body.append('Hello, ' + (user ? user.first_name : 'guest'));
</script>
```

!!! warning "`initDataUnsafe` is not trusted"
    Anything you do that matters (saving data, granting access) must verify the **signed**
    `KorfixApp.initData` on your backend. See [SDK reference & auth](sdk.md#validate-initdata-on-your-backend).

---

## 3. Add a main button and send a result

```html
<script src="https://YOUR-KORFIX-HOST/templates/def/chat/korfix-web-app.js"></script>
<script>
  KorfixApp.ready();
  KorfixApp.MainButton.setText('Send to chat').show().onClick(function () {
    KorfixApp.sendData(JSON.stringify({ picked: 42 }));
  });
</script>
```

`sendData()` posts the payload into the room as a message (max 4096 bytes) and closes the app window.
This is the natural "app → chat" bridge.

---

## 4. Register the app in a room

1. Open the chat, press the **ℹ️ info** button in the header.
2. Go to the **Apps** tab.
3. Fill in **name**, **URL**, **open mode**, and an optional **secret** (see below), then **Save**.

Permissions: **group** chats — owners/admins; **personal** chats — either participant.

### Open mode

| Mode | Behavior |
|------|----------|
| `popup` (default) | A floating window inside the page — drag by the header, resize from the corner. |
| `newtab` | Opens in a new browser tab. |

### The secret

If your app validates the user on its **backend**, set a **secret** when registering the app. Korfix
signs `initData` with this secret using the standard Telegram‑compatible scheme, and your backend
verifies it with the same secret. Leave it empty for trusted internal pages that don't need
verification (the app then opens without `initData`).

---

## Checklist

- [ ] SDK included from the Korfix host (or vendored)
- [ ] `KorfixApp.ready()` called
- [ ] Backend validates `KorfixApp.initData` if you trust the user identity
- [ ] App registered in the room with a secret (if backend‑validated)
- [ ] `sendData()` used where the app should return a result to the chat
