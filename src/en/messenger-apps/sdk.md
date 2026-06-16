# SDK reference & auth

> **← [Overview](index.md)** · [Build your first app](getting-started.md)

The SDK exposes one global, **`window.KorfixApp`** (also available as `window.Korfix.WebApp`).

---

## `window.KorfixApp`

| Member | Description |
|--------|-------------|
| `initData` | The **signed** init‑data string. Send it to your backend to verify. |
| `initDataUnsafe` | Parsed object: `user`, `chat_instance`, `chat_type`, `auth_date`. **Do not trust without verifying the signature.** |
| `version`, `platform`, `themeParams` | Environment info. |
| `ready()` | Tell the host the app is ready. |
| `expand()` | Request more height. |
| `close()` | Close the app window. |
| `sendData(str)` | Send a payload → becomes a chat message; the window closes. Max 4096 bytes. |
| `MainButton` | `.setText(t).show()` / `.hide()` / `.setParams({...})` / `.onClick(cb)`. |
| `BackButton` | Same shape as `MainButton`. |
| `showAlert(msg, cb)` / `showConfirm(msg, cb)` / `openPopup(params, cb)` | Native‑style dialogs rendered by the host. |
| `HapticFeedback.impactOccurred(style)` / `.notificationOccurred(type)` | No‑op in the browser; safe to call. |
| `onEvent(type, cb)` / `offEvent(type, cb)` | Events: `mainButtonClicked`, `backButtonClicked`, `viewportChanged`, `themeChanged`, `popupClosed`. |

---

## What `initData` contains

`initData` is a URL‑encoded query string (the same shape Telegram uses):

| Field | Value |
|-------|-------|
| `user` | JSON: `{ id, first_name, last_name, username, language_code, photo_url }` |
| `chat_instance` | Opaque room identifier |
| `chat_type` | `group` or `personal` |
| `auth_date` | Unix timestamp of signing (check freshness) |
| `hash` | Signature |

Each user who opens the app gets their **own** signed `initData`, so in a group the app always knows
exactly who launched it.

---

## Validate `initData` on your backend

Never trust `initDataUnsafe` on the client. Send `KorfixApp.initData` to your server and verify the
signature with the **secret** you set when registering the app in the room.

**Scheme (identical to Telegram WebApp):**
`secret_key = HMAC_SHA256(key="WebAppData", message=SECRET)`, then compare
`HMAC_SHA256(check_string, secret_key)` to `hash`. `check_string` is the `key=value` pairs (excluding
`hash`), sorted by key, joined with `\n`.

=== "Node.js"

    ```js
    const crypto = require('crypto');

    function verifyKorfixInitData(initData, secret) {
      const p = new URLSearchParams(initData);
      const hash = p.get('hash');
      p.delete('hash');
      const checkString = [...p.entries()]
        .sort(([a], [b]) => a.localeCompare(b))
        .map(([k, v]) => `${k}=${v}`)
        .join('\n');
      const secretKey = crypto.createHmac('sha256', 'WebAppData').update(secret).digest();
      const calc = crypto.createHmac('sha256', secretKey).update(checkString).digest('hex');
      return crypto.timingSafeEqual(Buffer.from(calc), Buffer.from(hash));
    }
    // const user = JSON.parse(new URLSearchParams(initData).get('user'));  // after verify
    ```

=== "PHP"

    ```php
    function verify_korfix_init_data(string $initData, string $secret): bool {
        parse_str($initData, $d);
        if (empty($d['hash'])) return false;
        $hash = $d['hash']; unset($d['hash']);
        ksort($d);
        $cs = implode("\n", array_map(fn($k) => "$k=$d[$k]", array_keys($d)));
        $secretKey = hash_hmac('sha256', $secret, 'WebAppData', true);  // key='WebAppData', msg=secret
        return hash_equals(hash_hmac('sha256', $cs, $secretKey), $hash);
    }
    ```

=== "Python"

    ```python
    import hmac, hashlib
    from urllib.parse import parse_qsl

    def verify_korfix_init_data(init_data: str, secret: str) -> bool:
        data = dict(parse_qsl(init_data))
        got = data.pop('hash', '')
        check = '\n'.join(f'{k}={data[k]}' for k in sorted(data))
        secret_key = hmac.new(b'WebAppData', secret.encode(), hashlib.sha256).digest()
        calc = hmac.new(secret_key, check.encode(), hashlib.sha256).hexdigest()
        return hmac.compare_digest(calc, got)
    ```

!!! tip "Same scheme as Telegram"
    This is exactly Telegram's `initData` validation — only the **secret** differs (the per‑app secret
    you set in Korfix instead of a Telegram bot token). So existing Telegram backends work with a
    one‑line change. See [Adapt a Telegram app](telegram-compat.md).
