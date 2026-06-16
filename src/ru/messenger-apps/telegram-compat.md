# Адаптация Telegram Mini App

Приложение, написанное под Telegram (использует `telegram-web-app.js` и `window.Telegram.WebApp`),
запускается в Korfix с **минимальными правками**.

> **← [Обзор](index.md)** · [SDK и авторизация](sdk.md)

---

## 1. Замените SDK‑скрипт

Замените скрипт Telegram на **адаптер совместимости** Korfix:

```html
<!-- было -->
<!-- <script src="https://telegram.org/js/telegram-web-app.js"></script> -->

<!-- стало -->
<script src="https://ВАШ-KORFIX-ХОСТ/templates/def/chat/korfix-web-app-tg-compat.js"></script>
```

Адаптер сам подгружает SDK Korfix (с того же хоста, откуда взят) и публикует `window.Telegram.WebApp`
как алиас `window.KorfixApp`. **Вы подключаете только этот один скрипт** — отдельно добавлять
`korfix-web-app.js` не нужно.

!!! tip "Если приложение на вашем домене"
    Адаптер грузит SDK относительно собственного URL, поэтому подключение адаптера с хоста Korfix
    работает, даже когда приложение отдаётся с другого домена. Можно также вендорить оба файла
    (`korfix-web-app.js` + `korfix-web-app-tg-compat.js`) рядом с приложением.

После этого ваши вызовы продолжают работать:

```js
const tg = window.Telegram.WebApp;
tg.ready();
tg.expand();
const user = tg.initDataUnsafe.user;
tg.MainButton.setText('Подтвердить').show().onClick(() => tg.sendData(JSON.stringify(payload)));
```

---

## 2. Наведите бэкенд на секрет Korfix

Схема подписи `initData` **идентична** Telegram
(`secret_key = HMAC_SHA256(key="WebAppData", message=secret)`), поэтому код валидации менять **не
нужно** — меняется только **значение секрета**:

- В Telegram секрет — это **токен бота**.
- В Korfix секрет — это **секрет приложения**, заданный при регистрации в комнате
  (ℹ️ информация → вкладка **Apps**).

Либо задайте секрет приложения в Korfix таким же, какой ожидает ваш бэкенд, либо пропишите в бэкенде
секрет, выданный в Korfix. Сниппеты проверки — в [SDK и авторизация](sdk.md).

---

## 3. Учтите различия

**Поля `initData`.** Korfix передаёт подмножество: `user`, `chat_instance`, `chat_type`, `auth_date`,
`hash`. Поля, специфичные для Telegram (`query_id`, `chat`, `start_param`, `signature`, `receiver`),
**не передаются** — приложениям, завязанным на них, нужен фолбэк.

**Покрытие API.** Адаптер отображает основную поверхность:

| Поддерживается | Пока не поддерживается |
|----------------|------------------------|
| `ready` / `expand` / `close` | `CloudStorage` |
| `sendData` | `requestContact` / `requestWriteAccess` |
| `MainButton` / `BackButton` | `openLink` / `openTelegramLink` |
| `showPopup` / `showAlert` / `showConfirm` | счета / платежи |
| `HapticFeedback` | QR‑сканер, геолокация, биометрия |
| `onEvent` / `offEvent`, `themeParams` | |

Вызовы неподдерживаемых методов — no‑op или `undefined`; добавьте проверки, если приложение на них
полагается.

---

## Чеклист переноса

- [ ] `telegram-web-app.js` заменён на `korfix-web-app-tg-compat.js` с хоста Korfix
- [ ] Бэкенд проверяет секретом приложения Korfix (схема та же, что была)
- [ ] Приложение корректно переживает отсутствие `query_id` / `start_param`
- [ ] Нет жёсткой зависимости от неподдерживаемых методов Bot API
- [ ] Приложение зарегистрировано в комнате с этим секретом
