# SDK и авторизация

> **← [Обзор](index.md)** · [Первое приложение](getting-started.md)

SDK создаёт один глобальный объект — **`window.KorfixApp`** (он же `window.Korfix.WebApp`).

---

## `window.KorfixApp`

| Член | Описание |
|------|----------|
| `initData` | **Подписанная** строка init‑data. Отправьте её на бэкенд для проверки. |
| `initDataUnsafe` | Разобранный объект: `user`, `chat_instance`, `chat_type`, `auth_date`. **Не доверять без проверки подписи.** |
| `version`, `platform`, `themeParams` | Сведения об окружении. |
| `ready()` | Сообщить хосту, что приложение готово. |
| `expand()` | Запросить больше высоты. |
| `close()` | Закрыть окно приложения. |
| `sendData(str)` | Отправить данные → станут сообщением чата; окно закроется. До 4096 байт. |
| `MainButton` | `.setText(t).show()` / `.hide()` / `.setParams({...})` / `.onClick(cb)`. |
| `BackButton` | Тот же интерфейс, что и `MainButton`. |
| `showAlert(msg, cb)` / `showConfirm(msg, cb)` / `openPopup(params, cb)` | Диалоги, отрисованные хостом. |
| `HapticFeedback.impactOccurred(style)` / `.notificationOccurred(type)` | В браузере — no‑op; вызывать безопасно. |
| `onEvent(type, cb)` / `offEvent(type, cb)` | События: `mainButtonClicked`, `backButtonClicked`, `viewportChanged`, `themeChanged`, `popupClosed`. |

---

## Что приходит в `initData`

`initData` — URL‑кодированная строка запроса (той же формы, что у Telegram):

| Поле | Значение |
|------|----------|
| `user` | JSON: `{ id, first_name, last_name, username, language_code, photo_url }` |
| `chat_instance` | Непрозрачный идентификатор комнаты |
| `chat_type` | `group` или `personal` |
| `auth_date` | Unix‑время подписи (проверяйте свежесть) |
| `hash` | Подпись |

Каждый пользователь, открывший приложение, получает **свою** подписанную `initData` — поэтому в группе
приложение всегда знает, кто именно его открыл.

---

## Проверка `initData` на бэкенде

Никогда не доверяйте `initDataUnsafe` на клиенте. Отправьте `KorfixApp.initData` на сервер и проверьте
подпись **секретом**, заданным при регистрации приложения в комнате.

**Схема (идентична Telegram WebApp):**
`secret_key = HMAC_SHA256(key="WebAppData", message=SECRET)`, затем сравнить
`HMAC_SHA256(check_string, secret_key)` с `hash`. `check_string` — пары `key=value` (кроме `hash`),
отсортированные по ключу, склеенные через `\n`.

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
    // const user = JSON.parse(new URLSearchParams(initData).get('user'));  // после проверки
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

!!! tip "Та же схема, что у Telegram"
    Это в точности проверка `initData` из Telegram — отличается лишь **секрет** (секрет приложения,
    заданный в Korfix, вместо токена Telegram‑бота). Поэтому готовые Telegram‑бэкенды работают с правкой
    в одну строку. См. [Адаптация Telegram‑приложения](telegram-compat.md).
