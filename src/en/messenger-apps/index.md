# Messenger Apps — Overview

**Messenger Apps** let you attach an external web app to a Korfix chat room (a one‑to‑one
conversation or a group). The app opens in an embedded, movable window right inside the chat,
receives a signed identity of the user who opened it, and can send a result back into the
conversation as a message.

If you have built a **Telegram Mini App** before, this will feel familiar — and you can port an
existing Telegram app with minimal changes (see [Adapt a Telegram app](telegram-compat.md)).

---

## How it works

- An app is just a **URL** plus a few settings, attached to a room.
- When the app is configured, an **app icon** appears to the left of the message input. Tapping it
  opens the app.
- The app loads the small **Korfix Web App SDK** and gets the launching user's identity via a
  signed `initData` string (passed in the URL hash, exactly like Telegram Mini Apps in a browser).
- The app talks to the chat through a `postMessage` bridge exposed as `window.KorfixApp`:
  show a main button, open dialogs, and — most importantly — **`sendData()`**, which turns the
  app's payload into a chat message and closes the window.

```
[ chat room ] --(app icon)--> [ app window (iframe) ]
       ^                                |
       |        sendData(payload)       v
   message  <----------------  KorfixApp bridge
```

---

## For users

Open a chat, press the **ℹ️ info** button in the header → **Apps** tab → add an app (name + URL).
In **group** chats only owners/admins can manage apps; in **personal** chats either participant can.
Once at least one app exists, the **app icon** shows up next to the message box for everyone in the room.

Apps open as a **floating window** you can drag and resize, or in a **new browser tab** — your choice
per app.

---

## For developers

| Page | What's inside |
|------|---------------|
| [Build your first app](getting-started.md) | Include the SDK, read the user, add a button, `sendData` |
| [SDK reference & auth](sdk.md) | `window.KorfixApp` API + how to validate `initData` on your backend |
| [Adapt a Telegram app](telegram-compat.md) | Run an existing Telegram Mini App in Korfix with minimal changes |

!!! note "Two kinds of Korfix apps — don't confuse them"
    **Messenger Apps** (this section) live inside chat rooms and use `window.KorfixApp`.
    **Marketplace miniapps** (the [Miniapps](../miniapps/index.md) section) embed into ERP catalogs and
    use `VMCRMUserApp`. Different surface, different SDK, different auth.
