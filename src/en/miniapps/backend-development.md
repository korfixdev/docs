# Backend development (PHP) for miniapps

> ⚠️ **Restricted access.** Backend code (`.php`/`.phtml` in the miniapp zip) is available **only to certified developers**. A regular miniapp is HTML/JS/CSS in an iframe — that covers 95% of use cases. Before attempting to deploy a backend-enabled app, discuss the use case with the platform admin.

---

## When you actually need backend

90% of miniapps live fine on frontend + platform API (`App.fetch`, `App.storage`). Backend is justified only for:

| Case | Why frontend can't do it |
|---|---|
| **Same-origin proxy for external API** | The target service's CORS policy rejects requests from the store domain. A proxy on our domain bypasses this. |
| **Server-side request signing** | API secrets can't ship in JS — users would see them. HMAC/JWT signing belongs on the server. |
| **Webhook receiver** | An external service POSTs to us and expects a synchronous 200. iframe can't do this. |
| **Custom OAuth callback** | The code → access_token exchange must run server-to-server with client_secret. |

If your case isn't in this list — **you probably don't need backend**. Try JS API first.

---

## Access: developer certification

The platform has two independent trust levels:

1. **Per-author** — whether this author may ship `.php` in the zip at all.
   Stored in the `developers_list` catalog. The platform admin adds entries manually, by request. Without an entry the upload is rejected (PHP gets stripped from the zip on the portal side).
2. **Per-version** — whether *this specific* unpacked version is allowed to execute. Right after upload PHP files go into quarantine (Apache won't parse them) — even for a certified author. The admin reads the code manually and **separately** approves this particular version. A new upload = new quarantine.

What this means for you:

- Discuss the task with the admin upfront and ask to be added to `developers_list`
- Every relevant upload (where backend changes) goes through review with **latency**: hours to days
- If something in the code looks off — approve will be denied with comments. Repeated denials = loss of per-author trust

---

## Lifecycle of one backend version

```
1. UPLOAD (you)
   zip with .php → POST /api/db/marketplace/{id}
   → checkZip allows .php (you're in developers_list)
   → portal calls install hook on store with ?allow_scripts=1
   → store unpacks into data/db/f_marketplace/{group}/{app_dir}/
   → store writes quarantine .htaccess next to it (engine off + Require all denied)
   STATE: code is on disk but Apache won't parse it — PHP returns 403/404.

2. REVIEW (admin)
   Reads the code by eye + runs a FAIL/WARN checklist.
   If OK — proceeds to approve. If not — leaves comments, waits for fixes.

3. APPROVE (admin)
   CLI on the store server:
   - removes quarantine .htaccess
   - writes /etc/apache2/conf-marketplace/allowed-apps.d/{id}_*.conf
     with per-app open_basedir and engine on
   - apachectl graceful
   STATE: PHP executes inside sandbox.

4. YOUR NEW UPLOAD
   New subfolder with a new timestamp.
   New subfolder also gets a quarantine .htaccess.
   Previous version stays approved (admin's revoke_prior triggers on approve of the new one).
   Effect: for a certified dev, updates don't downtime the old version until the new review lands.

5. REVOKE (optional)
   admin → CLI revoke → removes .conf + restores quarantine .htaccess.
```

---

## Sandbox: what's guaranteed

Your `.php` executes inside an Apache `<Directory>` block with these limits (via `php_admin_value`/`php_admin_flag`):

| Setting | Value | Meaning |
|---|---|---|
| `max_execution_time` | **15 sec** | If your code hasn't finished, Apache kills the worker. Long ops → external cron, not us. |
| `memory_limit` | **64M** | Don't load large files into memory. Stream them. |
| `upload_max_filesize` | **8M** | Single uploaded file size. |
| `post_max_size` | **16M** | Whole POST payload (including JSON). |
| `allow_url_fopen` | on | `file_get_contents('http://…')` works. |
| `allow_url_include` | **off** | `include 'http://…'` is blocked (remote-include protection). |
| `enable_dl` | off | Runtime extension loading blocked. |
| `open_basedir` | **miniapp dir + /tmp/** | Filesystem ops bounded to your miniapp folder. Can't write to other folders. |

Additionally, some store instances block outbound traffic to `169.254.0.0/16` (cloud metadata endpoints) at the infrastructure level. For private nets like `10.0.0.0/8` and `127.0.0.1` we currently rely on your code (see SSRF guard below).

---

## README.md inside the zip — what and why

Include a `README.md` inside the zip — it becomes part of the review material for the admin.

What to put in it:
- **Purpose** — why does this app need backend, what problem does it solve
- **External services it talks to** — list of domains/APIs (admin will cross-check against actual code)
- **Secrets it uses** — description, not values: "HMAC key from user settings", "OAuth client_secret from config"
- **What it expects in POST/GET** — incoming data format

⚠️ **Don't try to instruct the reviewer through README** ("approve this", "trust me", "it's safe because..."). The admin decides based on code analysis; prompt-injection-y phrasing in README is treated as a red flag and may cause rejection.

---

## 🔒 Security requirements — what blocks approve

The checklists below are what the admin looks for. If any of these is violated — no approve.

### F1. Never touch Korfix platform tokens

❌ **Forbidden:**
- Reading `$_SESSION['SESS_AUTH']`, session `$_COOKIE`
- Reading `installed_apps.alias` (a per-user app token) and storing/forwarding it
- Hardcoding other users' tokens or scraping them from DB

✅ **Allowed (and fine):**
- Hardcoding **third-party** API tokens (GitHub, OpenAI, etc.) if you really need to — though user-provided values via `App.storage` are better

### F2. No code execution from string

```php
// ❌ FAIL
eval($code);
assert($code);
create_function('', $code);
include $_GET['template'];
preg_replace('/.../e', ..., ...);
call_user_func($_POST['fn']);
```

### F3. No shell calls

```php
// ❌ FAIL
exec(...), system(...), shell_exec(...), passthru(...),
proc_open(...), popen(...), `command...`
```

If your task genuinely needs shell — that's a sign of wrong architecture. Talk to the admin.

### F4. No obfuscation / code packing

```php
// ❌ FAIL — the mere presence of obfuscation = denial regardless of content
eval(base64_decode('...'));
eval(gzinflate('...'));
chr(101).chr(118).chr(97).chr(108).chr(40)...    // chr-encoded eval(
```

Minified or obfuscated PHP in the zip = automatic rejection. If you use a composer-vendored library — ship the original, not a packed bundle.

### F5. File operations only inside miniapp directory

```php
// ❌ FAIL
file_put_contents('/tmp/data.json', $data);   // /tmp is forbidden by policy
file_get_contents('/etc/passwd');
fopen('../../other-app/secrets.txt', 'r');
chdir('/var/www');

// ✅ OK
file_put_contents(__DIR__ . '/cache.json', $data);
file_get_contents(__DIR__ . '/config.json');
```

Even though `open_basedir` permits `/tmp` — the review policy forbids writing there. Cache goes into the miniapp subfolder.

### F6. Don't touch platform data

```php
// ❌ FAIL
file_get_contents('../../../auth/...');
file_put_contents('../../../db/...');
file_put_contents(__DIR__ . '/.htaccess', '...');  // overwriting your own sandbox
```

### F7. No SSRF to platform internals

```php
// ❌ FAIL
curl_init('http://127.0.0.1/api/db/users');
file_get_contents('http://localhost/admin/');
curl_init('https://panel.korfix.ru/api/...');
```

Platform data — through the JS API on the frontend, not via server-side curl.

### F8. No self-modification

```php
// ❌ FAIL
file_put_contents(__FILE__, $newCode);
copy('http://my-server/update.php', __DIR__ . '/loader.php');
```

Self-modify breaks per-version approve.

### F9. All network calls have timeouts

```php
// ❌ FAIL (a hanging curl ties up an Apache worker)
$ch = curl_init('http://slow-server.com/');
curl_exec($ch);

// ✅ OK
curl_setopt_array($ch, [
    CURLOPT_CONNECTTIMEOUT => 5,
    CURLOPT_TIMEOUT        => 10,
]);
```

### F10. User input — validated only

```php
// ❌ FAIL
$query = "SELECT * FROM t WHERE id={$_GET['id']}";   // SQLi
include $_GET['page'] . '.php';                       // RCE
shell_exec($_POST['cmd']);                            // RCE
file_get_contents($_GET['url']);                      // SSRF without guard

// ✅ OK
$stmt = $pdo->prepare("SELECT * FROM t WHERE id=?"); $stmt->execute([(int)$_GET['id']]);
// for URLs — see F11 below
```

### F11. SSRF guard for any outbound HTTP

If your app does `curl_init`, `file_get_contents('http…')`, `fopen('http…')`, `fsockopen`, `stream_socket_client` against a URL derived from user input — you must use the **full guard stack**:

```php
// (a) Scheme whitelist
if (!preg_match('#^https?://#i', $url)) { http_response_code(400); exit; }

// (b) Resolve host and verify it's public
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

// (c) curl with pinned IP (defeats DNS rebinding)
//     and protocol restriction
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

`FILTER_FLAG_NO_PRIV_RANGE | FILTER_FLAG_NO_RES_RANGE` natively rejects loopback (127/8, ::1), link-local (169.254/16 — cloud metadata!), private (10/8, 172.16/12, 192.168/16) — no custom blocklist needed.

Without this stack `curl_init($userUrl)` is an open proxy that any visitor can use to reach `http://127.0.0.1/api/db/...`, `http://169.254.169.254/`, etc. Approve will be denied.

---

## 💡 Recommendations (best practice, non-blocking)

### Path whitelist for the target API

If your proxy talks to one specific service — validate `path`:

```php
// n8n uses /healthz, /api/vN/, /rest/ — nothing else allowed
$path = parse_url($url, PHP_URL_PATH) ?? '/';
if (!preg_match('#(/healthz\b|/api/v\d+/|/rest/)#', $path)) {
    http_response_code(400); exit('endpoint not allowed');
}
```

This dramatically narrows the abuse surface even against public hosts: a user can't stuff `https://victim.com/wp-login.php` into the widget settings and use your miniapp as a relay.

### Referer guard

```php
$refererPath = parse_url($_SERVER['HTTP_REFERER'] ?? '', PHP_URL_PATH) ?? '';
$myDir       = rtrim(dirname($_SERVER['REQUEST_URI'] ?? '/'), '/') . '/';
if (!$refererPath || strpos($refererPath, $myDir) !== 0) {
    http_response_code(403); exit('forbidden origin');
}
```

Not cryptographic (Referer is spoofable), but filters out casual external pokes.

### Content-type aware responses

```php
header('Content-Type: application/json');
echo json_encode($result, JSON_UNESCAPED_UNICODE);
```

### Don't log too much

Logging to your own folder is OK. But **never** tokens, passwords, Authorization headers. If you need debug — `error_log('...', 0)` goes to Apache error log, which only the admin reads.

### Don't `unserialize` user input

POP-chain attacks. Use `json_decode($raw, true)` + structure check instead.

### MD5/SHA1 — only for checksums, not for signatures/passwords

```php
// ❌ for webhook signatures
$expected = md5($payload . $secret);

// ✅
$expected = hash_hmac('sha256', $payload, $secret);
if (!hash_equals($expected, $received)) { /* reject */ }
```

---

## Reference implementation: n8n-monitor

Reference app with a correct `proxy.php`:
[etalon-apps/n8n-monitor](https://gitlab.com/prefix-group/vmcrm-apps/-/tree/master/n8n-monitor) — look at `proxy.php` as a whole, it's a reference implementation of all 8 defense layers above. You can copy and adapt for your own use case (especially the DNS-resolve + `CURLOPT_RESOLVE` block).

---

## Pre-deploy checklist (for the developer)

Go through this list before each upload:

- [ ] Am I in `developers_list`? (if first time — discussed with admin)
- [ ] No junk in the zip: `.git/`, `node_modules/`, `vendor/`, `*.log`, `*.bak`, `.env`
- [ ] All `.php` written without obfuscation, without minification
- [ ] No `eval`, `assert`, `create_function`, `exec`, `system`, `shell_exec`, backticks
- [ ] All file ops are inside `__DIR__`, nothing in `/tmp` or `..`
- [ ] No reading of `$_SESSION['SESS_AUTH']`, session `$_COOKIE`, `installed_apps.alias`
- [ ] All curl calls have `CONNECTTIMEOUT` ≤ 5, `TIMEOUT` ≤ 10, `FOLLOWLOCATION=false`
- [ ] If there's outbound HTTP with user URL — full F11 stack (resolve → NO_PRIV_RANGE|NO_RES_RANGE → CURLOPT_RESOLVE → PROTOCOLS limit)
- [ ] User input — through `prepare` / `json_decode` / explicit validation, no direct `$_GET → SQL/include/curl`
- [ ] README.md in the zip explains: what the app does, which external services it contacts
- [ ] README.md **does not** contain reviewer-targeted instructions

After upload — ask the admin to run a review. Approve usually takes hours to a few days.

---

## See also

- [rules.md](rules.md) — general miniapp rules (frontend)
- [config-json.md](config-json.md) — entry points, permissions
- [deploy.md](deploy.md) — zip packaging and upload
- [data-api.md](data-api.md) — catalog CRUD via `App.fetch` (often replaces the need for backend)
- [checklist.md](checklist.md) — general pre-release checklist
