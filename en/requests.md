# Requests & Responses

Every action receives a `Luany\Core\Http\Request` and produces a `Luany\Core\Http\Response`. This page lists both APIs.

## Accessing the request

Type-hint `Request` in a controller method or closure:

```php
use Luany\Core\Http\Request;

public function store(Request $request): Response
{
    $title = $request->input('title');
    // ...
}
```

The Kernel creates it with `Request::fromGlobals()`, which:

- applies **method spoofing**: a `POST` with a `_method` field of `PUT`, `PATCH` or `DELETE` becomes that method;
- strips the front controller's directory from the path, so apps in a sub-directory see `/posts`, not `/myapp/posts`;
- decodes **JSON bodies** when `Content-Type` contains `application/json`.

## Method and URI

| Method | Returns |
| --- | --- |
| `method()` | Upper-case method, after spoofing: `'GET'`, `'PUT'`, … |
| `isMethod('post')` | Case-insensitive comparison |
| `isGet()`, `isPost()`, `isPut()`, `isPatch()`, `isDelete()` | `bool` |
| `uri()` | Path without query string, e.g. `/posts/5` |
| `url()` | Absolute URL without query string: scheme, `Host` and path |
| `isAjax()` | `X-Requested-With: XMLHttpRequest` |
| `expectsJson()` | `Accept` contains `application/json`, or an AJAX request |

## Input

| Method | Description |
| --- | --- |
| `input($key, $default = null)` | Body value, then query value, then `$default` |
| `query($key, $default = null)` | Query string only |
| `post($key, $default = null)` | Body only (form fields or decoded JSON) |
| `body()` | The whole body as an array |
| `queryParams()` | The whole query string as an array |
| `all()` | Query and body merged (body wins) |
| `only(['a', 'b'])` / `except(['c'])` | Subset of `all()` |
| `has($key)` | Key present and not `null`, in body or query |
| `filled($key)` | Present and not `''` |

```php
$page   = (int) $request->query('page', 1);
$fields = $request->only(['title', 'body']);

if ($request->filled('search')) {
    // ...
}
```

> **Tip** Validate `$request->body()` rather than `all()` so query-string values cannot be smuggled into your data: `validate($request->body(), $rules)`.

## Files

| Method | Description |
| --- | --- |
| `file($key)` | The `$_FILES` entry (array) or `null` |
| `hasFile($key)` | An upload exists and has no error |
| `files()` | All uploaded files |

```php
if ($request->hasFile('avatar')) {
    $file = $request->file('avatar');
    move_uploaded_file($file['tmp_name'], base_path('storage/avatars/' . bin2hex(random_bytes(8))));
}
```

## Headers, cookies and server values

| Method | Description |
| --- | --- |
| `header($name, $default = null)` | Case-insensitive: `header('content-type')` |
| `headers()` | All headers |
| `cookie($name, $default = null)` | A cookie value |
| `hasCookie($name)` / `cookies()` | Presence check / all cookies |
| `server($key, $default = null)` / `serverParams()` | `$_SERVER` values |
| `ip()` | Client IP — honours `X-Forwarded-For` only from [trusted proxies](middleware.md#trusted-proxies) |
| `isSecure()` | HTTPS — honours `X-Forwarded-Proto` only from trusted proxies |
| `userAgent()` | `User-Agent` header |

## Creating responses

Return a string or array for simple cases; build a `Response` for full control.

```php
use Luany\Core\Http\Response;

return Response::make('<h1>Hello</h1>');
return Response::make('Created', 201, ['X-Id' => '42']);
return Response::json(['id' => 42], 201);
return Response::redirect('/login');
return Response::redirect('/new-home', 301);
```

| Factory | Default |
| --- | --- |
| `make($body = '', $status = 200, $headers = [])` | Plain response |
| `json($data, $status = 200, $headers = [], $flags)` | `Content-Type: application/json; charset=UTF-8`; unescaped slashes and Unicode |
| `redirect($url, $status = 302)` | `Location` header |
| `notFound()`, `unauthorized()`, `forbidden()`, `serverError()` | 404 / 401 / 403 / 500 with a minimal HTML body |

`json()` replaces invalid UTF-8 instead of returning an empty body; any other encoding error throws `JsonException`.

The `response()` and `redirect()` [helpers](helpers.md) are shortcuts for `make()` and `redirect()`.

### Fluent methods

```php
return Response::make(view('pages.report'))
    ->status(200)
    ->header('Cache-Control', 'no-store')
    ->withHeaders(['X-Frame-Options' => 'DENY']);
```

| Method | Description |
| --- | --- |
| `status($code)` | Set the status code |
| `body($string)` | Replace the body |
| `header($name, $value)` / `withHeaders([...])` | Set headers |
| `setProtocolVersion('2')` | HTTP version in the status line |
| `getStatusCode()`, `getBody()`, `getHeaders()`, `getCookies()` | Read back |
| `isRedirect()`, `isSuccessful()` | Status checks |

These methods modify the same response object and return it.

## Cookies

```php
return redirect('/')
    ->cookie('theme', 'dark', time() + 60 * 60 * 24 * 365, secure: $request->isSecure());
```

```php
cookie(
    string $name,
    string $value,
    int $expires = 0,          // Unix timestamp; 0 = session cookie
    string $path = '/',
    string $domain = '',
    bool $secure = false,
    bool $httpOnly = true,
    string $sameSite = 'Lax'   // Lax | Strict | None
)
```

Cookies are `HttpOnly` and `SameSite=Lax` by default. `SameSite=None` forces `Secure`. Invalid names or SameSite values throw `InvalidArgumentException`. Call `cookie()` several times to set several cookies.

| Method | Description |
| --- | --- |
| `forgetCookie($name, $path = '/', $domain = '')` | Expire a cookie |
| `rawCookie($setCookie)` | Append a pre-formatted `Set-Cookie` value |

Read cookies with `$request->cookie('theme')`.

## PSR-7

Both classes convert to and from PSR-7 messages:

```php
$psrRequest  = $request->toPsr();
$request     = Request::fromPsr($psrRequest);

$psrResponse = $response->toPsr();
$response    = Response::fromPsr($psrResponse);
```

This needs a PSR-17 factory: `nyholm/psr7` or `guzzlehttp/psr7` are detected automatically, or call `Luany\Core\Psr\Psr17::use($factory)`. See [PSR Interoperability](psr.md).
