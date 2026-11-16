# Security

Luany is secure by default where it can be: output is escaped, state-changing requests need a CSRF token, models refuse mass assignment until you allow it, and SQL identifiers are validated. This page summarises the built-in protections, what you must configure yourself, and a checklist for production.

## Built-in protections

| Threat | Protection | Default | Details |
|---|---|---|---|
| Cross-site scripting (XSS) | `{{ }}` escapes output | On | [Template Syntax](lte-syntax.md) |
| Cross-site request forgery | `CsrfMiddleware` checks `POST`, `PUT`, `PATCH`, `DELETE` | On in the skeleton | [CSRF Protection](csrf.md) |
| Mass assignment | Models are fully guarded until `$fillable` is set | On | [Models](models.md) |
| SQL injection | Values are bound; identifiers are validated and quoted | On | [Query Builder](query-builder.md) |
| Session hijacking / fixation | Hardened session cookie, strict mode, `regenerate()` | On | [Session & Flash](session.md) |
| Open redirects | Validation redirects only to same-host referers | On | [Validation](validation.md) |
| IP / HTTPS spoofing | Forwarded headers ignored unless the proxy is trusted | On | [Middleware](middleware.md#trusted-proxies) |
| Leaking errors | Debug pages only with `APP_DEBUG=true` | Depends on `.env` | [Error Handling](errors.md) |
| Cross-origin requests | `CorsMiddleware` | Opt-in | [Middleware](middleware.md#cors) |
| Brute force | `RateLimitMiddleware` | Opt-in | [Middleware](middleware.md#rate-limiting) |
| Exposed dotfiles, listings | `public/.htaccess` denies dotfiles and indexes | Apache only | [Deployment](deployment.md) |

## Output escaping

`{{ $value }}` is escaped with `htmlspecialchars()` using `ENT_QUOTES | ENT_SUBSTITUTE` and UTF-8, so quotes are safe inside attributes and invalid UTF-8 cannot blank the output. `{!! $value !!}` prints raw HTML.

```lte
<p>{{ $comment->body }}</p>          {{-- escaped --}}
<div>{!! $page->trustedHtml !!}</div> {{-- raw: only for HTML you control --}}
<script>const user = @json($user);</script>
```

> **Warning** Never use `{!! !!}` with user input. Use `@json` to pass data to JavaScript — it encodes HTML-significant characters. In PHP code, `e($value)` escapes the same way as `{{ }}`.

Custom directives return code that is **not** escaped; escape inside the generated code (see [Custom Directives](lte-custom.md)).

## CSRF

The skeleton registers `App\Http\Middleware\CsrfMiddleware` globally. Every `POST`, `PUT`, `PATCH` and `DELETE` request must carry the session token in the `_token` field (`@csrf`) or the `X-CSRF-TOKEN` header; tokens are compared with `hash_equals()`. A mismatch returns `419` (JSON for JSON clients).

```lte
<form method="POST" action="/profile">
    @csrf
    @method('PUT')
    ...
</form>
```

Exclude only endpoints that authenticate in another way, such as signed webhooks, with `$except = ['/webhooks/*']`.

## Mass assignment

A model with an empty `$fillable` is fully guarded (`$guarded = ['*']`). `fill()` and `update()` silently drop every attribute that is not fillable, and `create()` throws when nothing fillable remains — arbitrary request keys never reach the database. List the columns users may set:

```php
class User extends Model
{
    protected array $fillable = ['name', 'email', 'password'];
    protected array $hidden   = ['password'];
}
```

Combine it with validation — `validate()` returns only the validated fields, so `User::create($data)` never sees extra input such as `is_admin`.

## SQL injection

The query builder binds every value. Table and column names that reach SQL are validated against a strict identifier pattern and quoted for the driver, and operators are checked against a whitelist — invalid names throw `InvalidArgumentException` instead of being concatenated. The `unique` and `exists` validation rules validate and quote their table and column names the same way.

> **Warning** Raw SQL you write yourself (`$pdo->query()`, raw expressions) is not protected. Always use prepared statements with bound parameters.

## Sessions

The file session starts with secure cookie defaults:

| Option | Default |
|---|---|
| `httponly` | `true` — no JavaScript access |
| `samesite` | `Lax` |
| `secure` | Automatic on HTTPS (`$_SERVER['HTTPS']` or port 443) |
| `strict_mode` | `true` — uninitialised session IDs are rejected |
| `only_cookies` | `true` — no session IDs in URLs |

Regenerate the session ID and the CSRF token when the user's privilege changes (login, logout):

```php
session()->regenerate();
app('csrf')->regenerate();
```

> **Note** Behind a TLS-terminating proxy PHP does not see HTTPS, so the automatic `secure` flag stays off. Set `'secure' => true` in `config/session.php`.

Session files contain CSRF tokens and user data. The skeleton's `.gitignore` excludes `storage/sessions/` (since skeleton 1.4.2); in an older project, add that line and remove any committed session files with `git rm -r --cached storage/sessions`.

## Trusted proxies

`$request->ip()` and `$request->isSecure()` ignore `X-Forwarded-For` and `X-Forwarded-Proto` unless `REMOTE_ADDR` is a trusted proxy, so clients cannot spoof their address (which would defeat rate limiting). Behind a load balancer, trust it explicitly:

```php
use Luany\Core\Http\Request;

Request::setTrustedProxies(['10.0.0.0/8']);
```

## Safe redirects

When `validate()` fails without an explicit redirect URL, it redirects to the `Referer` only if it is a relative path or points to the current host; anything else (`//evil.com`, `https://evil.com`, `javascript:` URLs, credentials in the URL) falls back to `/`. Old input flashed back to forms never contains `password*`, `_token`, `csrf_token` or `_method`.

When you redirect to a URL taken from the request yourself, validate it the same way:

```php
use Luany\Framework\Support\Url;

$next   = $request->input('next');
$target = Url::sameHostOrFallback(is_string($next) ? $next : null, (string) $request->header('Host', ''));

return redirect($target);
```

## CORS and rate limiting

Both are opt-in middleware from `luany/core`:

```php
use Luany\Core\Middleware\CorsMiddleware;
use Luany\Core\Middleware\RateLimitMiddleware;
use Luany\Core\RateLimit\FileRateLimiter;

Route::post('/login', [AuthController::class, 'login'])
    ->middleware(new RateLimitMiddleware(
        new FileRateLimiter(base_path('storage/rate-limits')),
        maxAttempts: 5,
        decaySeconds: 60,
    ));

Route::middleware(new CorsMiddleware(
    allowedOrigins:   ['https://app.example.com'],
    allowCredentials: true,
))->group(function () {
    Route::get('/api/me', [ApiController::class, 'me']);
});
```

`CorsMiddleware` refuses the unsafe combination of `['*']` with `allowCredentials: true`. Rate-limited requests receive `429` with `Retry-After`.

## File uploads

Validate uploads with the `file`, `image` and `mimes` rules and a `max` size in kilobytes (see [Validation](validation.md#file-uploads)), generate the stored file name yourself, and store files outside `public/` unless they are meant to be public:

```php
$data = validate($request->body() + $request->files(), [
    'avatar' => 'required|mimes:png,jpg|max:2048',   // PNG or JPEG, up to 2 MB
]);

$file = $data['avatar'];
$mime = (new \finfo(FILEINFO_MIME_TYPE))->file($file['tmp_name']);
$name = bin2hex(random_bytes(16)) . ($mime === 'image/png' ? '.png' : '.jpg');

move_uploaded_file($file['tmp_name'], base_path('storage/uploads/' . $name));
```

The file rules are built to resist forged input:

- **`is_uploaded_file()`** — the `tmp_name` must be a file PHP received in this request. A JSON body or form field shaped like a `$_FILES` entry cannot make the validator read, accept or report on an arbitrary server file such as `/etc/passwd` or `.env`. Replace the check with `Validator::setUploadedFileVerifier()` only for tests or PSR-7 workers, and keep the replacement as strict as possible.
- **Content detection** — the type comes from the file's contents (`finfo`), not from the client's `Content-Type` header or file name. Keep the `fileinfo` extension enabled: without it the validator falls back to the client's file name.
- **No SVG in `image`** — SVG files can carry scripts. Accept them with `mimes:svg` only when you sanitise them or serve them with `Content-Disposition: attachment`.

A file that passes validation can still be harmful content; never store uploads under `public/` with a name or extension chosen by the client.

## Checklist

- [ ] `APP_ENV=production` and `APP_DEBUG=false` in production.
- [ ] `APP_KEY` set, and `.env` not readable by the web server's other users (`chmod 640`).
- [ ] The web server's document root is `public/` — never the project root.
- [ ] `CsrfMiddleware` is in the Kernel; `$except` contains only signed webhooks.
- [ ] Every model defines `$fillable`; sensitive columns are in `$hidden`.
- [ ] No `{!! !!}` with user-controlled data.
- [ ] Session ID and CSRF token regenerated on login and logout.
- [ ] HTTPS everywhere; `secure` session cookie and HSTS enabled.
- [ ] Trusted proxies configured when behind a load balancer.
- [ ] Rate limiting on login, registration and password reset routes.
- [ ] Uploads validated with `file` / `image` / `mimes` and `max`, and stored outside `public/`.
- [ ] `storage/sessions/` and `.env` not in version control.
- [ ] Dependencies up to date (`composer audit`).

See [Deployment](deployment.md) for the server side.
