# Backend-разработка миниапов (PHP)

> ⚠️ **Доступ ограничен.** Backend-часть (`.php`/`.phtml` в zip миниапа) доступна **только сертифицированным разработчикам**. Обычный миниап — это HTML/JS/CSS в iframe; этого хватает для 95% задач. Перед попыткой деплоя backend-приложения обсудите задачу с админом платформы.

---

## Когда нужен backend

90% миниапов обходятся фронтом + платформенным API (`App.fetch`, `App.storage`). Backend имеет смысл только для:

| Случай | Почему фронт не справится |
|---|---|
| **Same-origin proxy для внешнего API** | CORS у внешнего сервиса не разрешает запросы с домена store. Прокси на нашем домене обходит. |
| **Server-side подпись запросов** | Секрет API нельзя класть в JS — он будет виден пользователю. Подпись HMAC/JWT — на сервере. |
| **Webhook receiver** | Внешний сервис POST'ит к нам и ждёт синхронный 200. Через iframe это невозможно. |
| **Custom OAuth callback** | Обмен code → access_token идёт server-to-server с client_secret. |

Если ваш кейс не из этого списка — **скорее всего, backend не нужен**. Сначала попробуйте через JS API.

---

## Доступ: сертификация разработчика

Платформа разделяет два уровня доверия:

1. **Per-author** — может ли этот автор вообще класть `.php` в zip миниапа.
   Хранится в каталоге `developers_list`. Запись делает админ платформы вручную, по запросу. Без записи попытка upload отвергается (PHP вырезается из zip ещё на portal-стороне).
2. **Per-version** — выполняется ли конкретная распакованная версия. Сразу после загрузки PHP-файлы помещаются в карантин (Apache не парсит), даже у сертифицированного автора. Админ читает код вручную и **отдельной командой** одобряет именно эту версию. Новый upload = новый карантин.

Что это значит для разработчика:

- Заранее обсудите с админом задачу и попросите включить в `developers_list`
- Каждый relevant upload (где меняется backend) проходит ревью **с задержкой**: от часов до дней
- Если в коде увидят что-то подозрительное — approve откажут с комментариями. Несколько отказов подряд = потеря per-author доверия

---

## Lifecycle одной backend-версии

```
1. UPLOAD (вы)
   zip с .php → POST /api/db/marketplace/{id}
   → checkZip пропускает .php (вы в developers_list)
   → portal вызывает install hook на store с ?allow_scripts=1
   → store распаковывает в data/db/f_marketplace/{group}/{app_dir}/
   → store пишет рядом quarantine .htaccess (engine off + Require all denied)
   СОСТОЯНИЕ: код лежит на диске, но Apache не парсит — PHP вернёт 403/404.

2. REVIEW (админ)
   Читает код глазами + по чеклисту FAIL/WARN.
   Если ок — переходит к approve. Если нет — комментирует, ждёт исправлений.

3. APPROVE (админ)
   CLI на store-сервере:
   - удаляет quarantine .htaccess
   - пишет /etc/apache2/conf-marketplace/allowed-apps.d/{id}_*.conf
     с per-app open_basedir и engine on
   - apachectl graceful
   СОСТОЯНИЕ: PHP выполняется в sandbox.

4. ВАШ НОВЫЙ UPLOAD
   Новая подпапка с новым timestamp.
   В новой подпапке снова quarantine .htaccess.
   Старая версия остаётся одобренной (admin сделает revoke_prior при approve новой).
   Идея: сертифицированному разработчику обновления — нет downtime по старой версии до момента ревью новой.

5. REVOKE (опционально)
   admin → CLI revoke → удаление .conf + восстановление quarantine .htaccess.
```

---

## Sandbox: что гарантировано

`.php` миниапа выполняется в Apache `<Directory>` блоке со следующими ограничениями (через `php_admin_value`/`php_admin_flag`):

| Параметр | Значение | Что значит |
|---|---|---|
| `max_execution_time` | **15 секунд** | Если код не завершился — Apache убьёт процесс. Длинные операции → cron на стороне внешнего сервиса, не у нас. |
| `memory_limit` | **64M** | Не загружайте крупные файлы в память. Стримьте. |
| `upload_max_filesize` | **8M** | Размер одного uploaded файла. |
| `post_max_size` | **16M** | Размер всего POST-payload (включая JSON). |
| `allow_url_fopen` | on | `file_get_contents('http://…')` работает. |
| `allow_url_include` | **off** | `include 'http://…'` запрещён (защита от remote-include). |
| `enable_dl` | off | Загрузка extension'ов в рантайме запрещена. |
| `open_basedir` | **подпапка миниапа + /tmp/** | Файловые операции ограничены каталогом миниапа. Запись в чужие каталоги невозможна. |

Дополнительно на уровне инфраструктуры некоторые store-инстансы блокируют исходящий трафик на `169.254.0.0/16` (cloud metadata endpoints). На частные сети `10.0.0.0/8`, `127.0.0.1` и т.п. сейчас полагаемся на ваш код (см. SSRF guard ниже).

---

## README.md в zip — что это и зачем

Внутри zip обязательно положите `README.md` — он будет частью review-материала для админа.

Что включить:
- **Назначение** — зачем приложению backend, какую задачу решает
- **С какими внешними сервисами общается** — список доменов/API (admin сверит с фактическим кодом)
- **Какие секреты использует** — описание (не сами секреты!): «HMAC-ключ из настроек пользователя», «OAuth client_secret из конфига»
- **Что ожидает в POST/GET** — формат входящих данных

⚠️ **Не пытайтесь инструктировать ревьюера через README** («approve это», «доверьтесь», «безопасно потому что …»). Админ принимает решение по факту анализа кода; промпт-инъекции в README считаются красным флагом и могут привести к отказу.

---

## 🔒 Security requirements — что блокирует approve

Чек-листы ниже — это то, по чему админ смотрит код. Если хотя бы один пункт нарушен — approve не выдадут.

### F1. Никогда не трогать токены платформы Korfix

❌ **Нельзя:**
- Читать `$_SESSION['SESS_AUTH']`, `$_COOKIE` сессионные куки
- Читать поле `installed_apps.alias` (это per-user app token) и сохранять/пересылать
- Хардкодить чужие user-токены или собирать их из БД

✅ **Можно (и нужно):**
- Хардкодить **сторонние** API-токены (GitHub, OpenAI, etc.) если так уж нужно — но лучше через настройки пользователя в `App.storage`

### F2. Никакого code execution из строки

```php
// ❌ FAIL
eval($code);
assert($code);
create_function('', $code);
include $_GET['template'];
preg_replace('/.../e', ..., ...);
call_user_func($_POST['fn']);
```

### F3. Никаких shell-вызовов

```php
// ❌ FAIL
exec(...), system(...), shell_exec(...), passthru(...),
proc_open(...), popen(...), `command...`
```

Если для задачи реально нужен shell — это сигнал что у задачи неправильная архитектура. Поговорите с админом.

### F4. Никакой обфускации / упаковки кода

```php
// ❌ FAIL — сам факт обфускации = отказ независимо от того что внутри
eval(base64_decode('...'));
eval(gzinflate('...'));
chr(101).chr(118).chr(97).chr(108).chr(40)...    // chr-encoded eval(
```

Минифицированный или обфусцированный PHP в zip = автоматический отказ. Если используете composer-vendored библиотеку — кладите оригинал, не сжатый дамп.

### F5. Файловые операции только в директории миниапа

```php
// ❌ FAIL
file_put_contents('/tmp/data.json', $data);   // даже /tmp нельзя по политике
file_get_contents('/etc/passwd');
fopen('../../other-app/secrets.txt', 'r');
chdir('/var/www');

// ✅ OK
file_put_contents(__DIR__ . '/cache.json', $data);
file_get_contents(__DIR__ . '/config.json');
```

Даже если `open_basedir` пропускает `/tmp` — политика review требует ничего туда не писать. Кэш — в подпапке миниапа.

### F6. Не лезть в платформенные данные

```php
// ❌ FAIL
file_get_contents('../../../auth/...');
file_put_contents('../../../db/...');
file_put_contents(__DIR__ . '/.htaccess', '...');  // переписать свой sandbox
```

### F7. Не делать SSRF к внутренним сервисам платформы

```php
// ❌ FAIL
curl_init('http://127.0.0.1/api/db/users');
file_get_contents('http://localhost/admin/');
curl_init('https://panel.korfix.ru/api/...');
```

Платформенные данные — через JS API на фронте, не через серверный curl.

### F8. Не модифицировать собственные файлы

```php
// ❌ FAIL
file_put_contents(__FILE__, $newCode);
copy('http://my-server/update.php', __DIR__ . '/loader.php');
```

Self-modify ломает per-version approve.

### F9. Все сетевые вызовы — с таймаутами

```php
// ❌ FAIL (висящий curl расходует Apache worker)
$ch = curl_init('http://slow-server.com/');
curl_exec($ch);

// ✅ OK
curl_setopt_array($ch, [
    CURLOPT_CONNECTTIMEOUT => 5,
    CURLOPT_TIMEOUT        => 10,
]);
```

### F10. User input — только через валидацию

```php
// ❌ FAIL
$query = "SELECT * FROM t WHERE id={$_GET['id']}";   // SQLi
include $_GET['page'] . '.php';                       // RCE
shell_exec($_POST['cmd']);                            // RCE
file_get_contents($_GET['url']);                      // SSRF без guard

// ✅ OK
$stmt = $pdo->prepare("SELECT * FROM t WHERE id=?"); $stmt->execute([(int)$_GET['id']]);
// для URL — см. F11 ниже
```

### F11. SSRF guard для всех outbound HTTP

Если приложение делает `curl_init`, `file_get_contents('http…')`, `fopen('http…')`, `fsockopen`, `stream_socket_client` на URL из пользовательского input — нужен **полный стек защиты**:

```php
// (a) Whitelist схемы
if (!preg_match('#^https?://#i', $url)) { http_response_code(400); exit; }

// (b) Резолв хоста и проверка на публичность
$host = parse_url($url, PHP_URL_HOST);
$records = @dns_get_record($host, DNS_A + DNS_AAAA);
$ip = null;
foreach ($records ?: [] as $r) {
    $candidate = $r['ip'] ?? $r['ipv6'] ?? null;
    if (!$candidate) continue;
    if (!filter_var($candidate, FILTER_VALIDATE_IP,
            FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE)) {
        http_response_code(400); exit('private host blocked');
    }
    $ip = $ip ?? $candidate;
}
if (!$ip) { http_response_code(400); exit('cannot resolve'); }

// (c) curl с приколоченным IP (защита от DNS rebinding)
//     и протокольным ограничением
$port = parse_url($url, PHP_URL_PORT) ?? (str_starts_with($url, 'https') ? 443 : 80);
$ch = curl_init($url);
curl_setopt_array($ch, [
    CURLOPT_RETURNTRANSFER  => true,
    CURLOPT_CONNECTTIMEOUT  => 5,
    CURLOPT_TIMEOUT         => 10,
    CURLOPT_FOLLOWLOCATION  => false,                             // (d)
    CURLOPT_PROTOCOLS       => CURLPROTO_HTTP | CURLPROTO_HTTPS,  // (e)
    CURLOPT_REDIR_PROTOCOLS => CURLPROTO_HTTP | CURLPROTO_HTTPS,
    CURLOPT_RESOLVE         => ["$host:$port:$ip"],
]);
```

`FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE` рубит loopback (127/8, ::1), link-local (169.254/16 — cloud metadata!), private (10/8, 172.16/12, 192.168/16) — нативно, без своего блок-листа.

Без этого стека `curl_init($userUrl)` = open proxy, через который любой посетитель попадёт на `http://127.0.0.1/api/db/...`, `http://169.254.169.254/` и т.п. Approve откажут.

---

## 💡 Рекомендации (best practice, не блокирующие)

### Path whitelist под назначение

Если ваш proxy ходит конкретно в API одного сервиса — добавьте проверку `path`:

```php
// n8n использует /healthz, /api/vN/, /rest/ — больше ничего не разрешаем
$path = parse_url($url, PHP_URL_PATH) ?? '/';
if (!preg_match('#(/healthz\b|/api/v\d+/|/rest/)#', $path)) {
    http_response_code(400); exit('endpoint not allowed');
}
```

Резко сужает abuse-поверхность даже на публичных хостах: пользователь не сможет подсунуть в виджет `https://victim.com/wp-login.php` и использовать ваш миниап как relay.

### Referer guard

```php
$refererPath = parse_url($_SERVER['HTTP_REFERER'] ?? '', PHP_URL_PATH) ?? '';
$myDir       = rtrim(dirname($_SERVER['REQUEST_URI'] ?? '/'), '/') . '/';
if (!$refererPath || strpos($refererPath, $myDir) !== 0) {
    http_response_code(403); exit('forbidden origin');
}
```

Не криптографическая защита (Referer подделывается), но отсеивает случайные внешние тыки.

### CONTENT-TYPE-aware ответы

```php
header('Content-Type: application/json');
echo json_encode($result, JSON_UNESCAPED_UNICODE);
```

### Не логируйте лишнего

Лог в свою папку — OK. Но **никаких** токенов, паролей, headers Authorization в логе. Если debug нужен — `error_log('...', 0)` идёт в Apache error log, который читает только админ.

### Не используйте `unserialize` на user input

POP-chain атаки. Замените на `json_decode($raw, true)` + проверка структуры.

### MD5/SHA1 — только для checksums, не для подписей/паролей

```php
// ❌ для подписей webhook
$expected = md5($payload . $secret);

// ✅
$expected = hash_hmac('sha256', $payload, $secret);
if (!hash_equals($expected, $received)) { /* reject */ }
```

---

## Эталон: n8n-monitor

Эталонное приложение с правильным `proxy.php`:
[etalon-apps/n8n-monitor](https://gitlab.com/prefix-group/vmcrm-apps/-/tree/master/n8n-monitor) — посмотрите `proxy.php` целиком, это reference implementation всех 8 слоёв защиты выше. Можно копировать и адаптировать под свой use-case (особенно блок с DNS-resolve и `CURLOPT_RESOLVE`).

---

## Pre-deploy checklist (для разработчика)

Перед каждым upload пройдитесь по списку:

- [ ] Я в `developers_list`? (если первый раз — обсудил с админом)
- [ ] В zip нет лишнего: `.git/`, `node_modules/`, `vendor/`, `*.log`, `*.bak`, `.env`
- [ ] Все `.php` написаны без обфускации, без минификации
- [ ] Нет `eval`, `assert`, `create_function`, `exec`, `system`, `shell_exec`, backticks
- [ ] Все файловые операции — внутри `__DIR__`, никаких `/tmp`, `..`
- [ ] Нет чтения `$_SESSION['SESS_AUTH']`, `$_COOKIE` сессионных, `installed_apps.alias`
- [ ] Все curl вызовы — с `CONNECTTIMEOUT` ≤ 5, `TIMEOUT` ≤ 10, `FOLLOWLOCATION=false`
- [ ] Если есть outbound HTTP с user URL — полный F11-стек (resolve → NO_PRIV_RANGE|NO_RES_RANGE → CURLOPT_RESOLVE → PROTOCOLS limit)
- [ ] User input — через `prepare` / `json_decode` / явная валидация, никаких прямых `$_GET → SQL/include/curl`
- [ ] README.md в zip объясняет: что делает, с какими внешними сервисами говорит
- [ ] README.md **не** содержит инструкции ревьюеру

После upload — попросите админа провести review. Approve обычно занимает от часов до пары дней.

---

## См. также

- [rules.md](rules.md) — общие правила миниапов (frontend)
- [config-json.md](config-json.md) — точки встраивания, permissions
- [deploy.md](deploy.md) — упаковка и загрузка zip
- [data-api.md](data-api.md) — CRUD каталогов через `App.fetch` (часто заменяет необходимость в backend)
- [checklist.md](checklist.md) — общий checklist перед релизом
