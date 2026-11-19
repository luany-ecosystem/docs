# Helpers

luany/framework defines a set of global functions for everyday tasks. Each is declared only if a function with the same name does not exist yet, so an application can define its own version first.

## Overview

| Area | Helpers |
| --- | --- |
| Application | [`app`](#app), [`base_path`](#base_path), [`config`](#config), [`env`](#env) |
| HTTP | [`abort`](#abort), [`redirect`](#redirect), [`response`](#response) |
| URLs | [`url`](#url), [`route`](#route), [`asset`](#asset) |
| Views | [`view`](#view), [`e`](#e) |
| Session | [`session`](#session), [`flash`](#flash), [`get_flash`](#get_flash), [`old`](#old) |
| Security | [`csrf_token`](#csrf_token), [`csrf_field`](#csrf_field) |
| Validation | [`validate`](#validate) |
| Localization | [`__`](#__), [`locale`](#locale) |
| Logging | [`logger`](#logger) |

## Application

### `app`

`app(?string $abstract = null): mixed` — the application container, or a service resolved from it.

```php
app();                        // Luany\Framework\Application
app('session');               // a binding by key
app(ReportService::class);    // a class, autowired
```

See [Service Container](container.md).

### `base_path`

`base_path(string $path = ''): string` — absolute path inside the project.

```php
base_path();                      // /var/www/my-app
base_path('storage/exports');     // /var/www/my-app/storage/exports
```

### `config`

`config(string|array|null $key = null, mixed $default = null): mixed`

```php
config('app.name');                   // read
config('app.missing', 'fallback');    // with default
config(['app.debug' => false]);       // set (returns null)
config();                             // the Config repository
```

See [Configuration](configuration.md#accessing-configuration).

### `env`

`env(string $key, mixed $default = null): mixed` — an environment variable. `true`, `false`, `null` and `empty` are converted. See [Configuration](configuration.md#reading-values).

```php
env('APP_ENV', 'production');
```

## HTTP

### `abort`

`abort(int $code, string $message = '', array $headers = []): never` — stop the request with an HTTP error. See [Error Handling](errors.md#aborting-a-request).

```php
abort(404);
abort(403, 'Forbidden');
abort(429, 'Slow down', ['Retry-After' => '60']);
```

### `redirect`

`redirect(string $url, int $status = 302): Response`

```php
return redirect('/dashboard');
return redirect(route('posts.show', ['id' => $post->id]));
return redirect('/new-url', 301);
```

### `response`

`response(string $body = '', int $status = 200): Response`

```php
return response('<h1>Created</h1>', 201);
return response()->status(204);
```

For JSON use `Response::json()` or return an array. See [Requests & Responses](requests.md#creating-responses).

## URLs

URLs are built from `config('app.url')`, falling back to `APP_URL`.

### `url`

`url(string $path = ''): string`

```php
url();             // http://localhost:8000
url('/posts/1');   // http://localhost:8000/posts/1
```

### `route`

`route(string $name, array $params = []): string` — URL of a [named route](routing.md#named-routes). Parameters missing from the pattern become the query string; an unknown name throws a `RuntimeException`.

```php
route('posts.show', ['id' => 5]);           // http://localhost:8000/posts/5
route('posts.index', ['page' => 2]);        // http://localhost:8000/posts?page=2
```

### `asset`

`asset(string $path): string` — path of a file in `public/assets`, relative to the site root: `asset('css/app.css')` gives `/assets/css/app.css` (or `/app/assets/css/app.css` when `APP_URL` has a sub-folder). Pages keep their CSS and JavaScript on any host or port — a second `luany serve`, a LAN address, a proxy.

Set `ASSET_URL` (or `app.asset_url`) for an absolute base, e.g. a CDN or links in e-mails: `ASSET_URL=https://cdn.example.com` gives `https://cdn.example.com/assets/css/app.css`. Requires luany/framework 1.5; earlier versions prefix `APP_URL`.

```lte
<link rel="stylesheet" href="{{ asset('css/app.css') }}">
```

## Views

### `view`

`view(string $name, array $data = []): string` — render an LTE template to a string.

```php
return view('pages.posts.index', ['posts' => $posts]);
```

### `e`

`e(mixed $value): string` — HTML-escape a value, as `{{ }}` does in LTE.

```php
echo '<p>' . e($comment->body) . '</p>';
```

## Session

### `session`

`session(?string $key = null, mixed $default = null): mixed` — the session, or a value from it. To write, use the session object.

```php
session('user_id');
session('theme', 'light');
session()->set('theme', 'dark');
```

See [Session & Flash](session.md).

### `flash`

`flash(string $type, string $message): void` — a message for the next request.

```php
flash('success', 'Post saved.');
```

### `get_flash`

`get_flash(): ?array` — the message flashed on the previous request, as `['type' => ..., 'message' => ...]`, or `null`.

### `old`

`old(string $key, mixed $default = null): mixed` — input flashed by a failed validation.

```lte
<input name="title" value="{{ old('title', $post->title) }}">
```

## Security

### `csrf_token`

`csrf_token(): string` — the session's CSRF token.

```lte
<meta name="csrf-token" content="{{ csrf_token() }}">
```

### `csrf_field`

`csrf_field(): string` — a hidden `_token` input. In LTE, `@csrf` does the same.

```php
<form method="POST"><?= csrf_field() ?> ...</form>
```

See [CSRF Protection](csrf.md).

## Validation

### `validate`

`validate(array $data, array $rules, string $redirectTo = ''): array` — returns the validated fields, or throws a `ValidationException` that redirects back (or answers 422 JSON).

```php
$data = validate($request->body(), [
    'title' => 'required|string|max:255',
]);
```

See [Validation](validation.md).

## Localization

### `__`

`__(string $key, array $replace = []): string` — translate a key.

```php
__('welcome.title', ['name' => 'Ana']);
```

### `locale`

`locale(): string` — the active locale code.

See [Localization](localization.md).

## Logging

### `logger`

`logger(string|\Stringable|null $message = null, array $context = [], string $level = 'debug'): ?LoggerInterface` — write a record, or get the logger when called without a message.

```php
logger('Import started');
logger('Low stock for {sku}', ['sku' => 'A-1'], 'warning');
logger()->error('Payment failed', ['order' => 7]);
```

See [Logging](logging.md).
