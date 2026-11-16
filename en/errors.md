# Error Handling

Every exception thrown while handling a request is caught by the Kernel and turned into a response. Expected HTTP outcomes (404, 403, 422…) get the right status; unexpected errors are logged and shown as a 500 page — or as a detailed debug page in development.

## How exceptions become responses

| Exception | JSON client | Browser |
| --- | --- | --- |
| `ValidationException` | 422 `{"message", "errors"}` | Redirect back, errors flashed — see [Validation](validation.md) |
| `HttpException` (`abort()`) | `{"message": "…"}` with its status and headers | Rendered by the Handler |
| No route matched | 404 `{"message": "Not Found"}` | Rendered by the Handler |
| Wrong method | 405 with `Allow` header | Rendered by the Handler |
| `ModelNotFoundException` (`findOrFail()`, `firstOrFail()`) | 404 `{"message": "Not Found"}` | 404, rendered by the Handler |
| Anything else | Reported, then `Handler::renderJson()` | Reported, then `Handler::render()` |

A "JSON client" is a request whose `Accept` header contains `application/json`, or an AJAX request (`X-Requested-With: XMLHttpRequest`).

## Aborting a request

`abort()` stops the request with an HTTP error from anywhere in your code:

```php
abort(404);
abort(403, 'You cannot edit this post.');
abort(429, 'Slow down.', ['Retry-After' => '60']);
```

It throws `Luany\Framework\Exceptions\HttpException`; you can also throw it directly:

```php
use Luany\Framework\Exceptions\HttpException;

throw new HttpException(409, 'This slug is already taken.');
```

Without a message, a standard reason is used: `Bad Request`, `Unauthorized`, `Forbidden`, `Not Found`, `Method Not Allowed`, `Unprocessable Content`, `Too Many Requests`, `Internal Server Error`, or `HTTP Error` for other codes. Messages are HTML-escaped when rendered, so they may safely contain user input.

## The exception handler

`app/Exceptions/Handler.php` extends `Luany\Framework\Exceptions\Handler` and is bound in `bootstrap/app.php`, with debug mode taken from `APP_DEBUG`:

```php
$app->singleton(
    \Luany\Framework\Exceptions\Handler::class,
    fn() => new App\Exceptions\Handler((bool) Env::get('APP_DEBUG', false))
);
```

For each exception the Kernel calls `report()`, then `render()` — or [`renderJson()`](#json-apis) when the request expects JSON.

### Rendering

The base `render()` method produces:

| Status | Debug off | Debug on |
| --- | --- | --- |
| 4xx | `<h1>404 — Not Found</h1>` | Same |
| 5xx | `<h1>500 — Server Error</h1>` | Debug page: exception class, message, file, line, request and stack trace |

The skeleton's Handler renders your own views instead:

| Status | View |
| --- | --- |
| 404 | `views/errors/404.lte` |
| 5xx | `views/errors/500.lte` (the debug page when `APP_DEBUG=true`) |
| Other 4xx | Falls back to the base handler |

Edit those views freely — they extend the main layout. To add pages for other statuses, look the view up by status code:

```php
public function render(\Throwable $e): Response
{
    $status = $this->statusCodeFor($e);

    if ($status >= 500 && $this->debug) {
        return parent::render($e);
    }

    $view = is_file(app()->viewsPath("errors/{$status}.lte"))
        ? "errors.{$status}"
        : ($status >= 500 && is_file(app()->viewsPath('errors/500.lte')) ? 'errors.500' : null);

    if ($view === null) {
        return parent::render($e);
    }

    return Response::make(view($view, ['exception' => $e]), $status)
        ->withHeaders($this->headersFor($e));
}
```

Then create `views/errors/403.lte`, `views/errors/419.lte` and so on.

| Helper method | Returns |
| --- | --- |
| `statusCodeFor($e)` | `HttpException` status, 422 for validation, 404 / 405 for routing, 404 for `ModelNotFoundException`, otherwise 500 |
| `headersFor($e)` | `Allow` for 405, the `HttpException` headers, otherwise `[]` |

> **Warning** Never enable `APP_DEBUG` in production. The debug page reveals file paths, the exception message and the stack trace.

### Reporting

`report()` logs the exception through the [logger](logging.md) at `error` level, with the exception (and its trace) in the context. Exceptions whose class is listed in `$dontReport` are skipped. The framework's default list is:

```php
protected array $dontReport = [
    \Luany\Framework\Exceptions\HttpException::class,
    \Luany\Framework\Exceptions\ValidationException::class,
    \Luany\Core\Exceptions\RouteNotFoundException::class,
    \Luany\Core\Exceptions\MethodNotAllowedException::class,
];
```

luany/database's `ModelNotFoundException` is never reported either: a missing record is a 404, not an error.

To skip more exceptions, redeclare the list in your Handler and keep the defaults:

```php
protected array $dontReport = [
    \Luany\Framework\Exceptions\HttpException::class,
    \Luany\Framework\Exceptions\ValidationException::class,
    \Luany\Core\Exceptions\RouteNotFoundException::class,
    \Luany\Core\Exceptions\MethodNotAllowedException::class,
    \App\Exceptions\PaymentDeclinedException::class,
];
```

> **Note** Projects created from a skeleton older than 1.4.2 declare `protected array $dontReport = [];` in `app/Exceptions/Handler.php`, which replaces the default list and logs every 404 and `abort()` as an error. Delete that line.

Override `report()` to send exceptions elsewhere:

```php
public function report(\Throwable $e): void
{
    if (!$this->shouldReport($e)) {
        return;
    }

    parent::report($e);               // still write the log
    \Sentry\captureException($e);
}
```

Exceptions thrown inside `report()` are ignored, so a broken reporter never hides the original error page.

## JSON APIs

Requests that expect JSON never receive an HTML error page. HTTP and validation errors use the Kernel's JSON shape shown [above](#how-exceptions-become-responses); every other exception is reported and then rendered by `Handler::renderJson()`:

| Status | Debug off | Debug on |
| --- | --- | --- |
| 4xx | `{"message": "Not Found"}` (the `abort()` message or the reason phrase) | Adds `exception`, `file`, `line`, `trace` |
| 5xx | `{"message": "Server Error"}` | Real message, plus `exception`, `file`, `line`, `trace` |

```json
{
    "message": "Server Error"
}
```

The status and headers come from `statusCodeFor()` and `headersFor()`. Override `renderJson()` to change the shape for your API:

```php
public function renderJson(\Throwable $e): Response
{
    return Response::json([
        'error' => [
            'status'  => $this->statusCodeFor($e),
            'message' => $this->debug ? $e->getMessage() : 'Something went wrong.',
        ],
    ], $this->statusCodeFor($e), $this->headersFor($e));
}
```

## When the handler itself fails

If no Handler is bound, or `render()` / `renderJson()` throws, the Kernel falls back to plain responses: the HTTP exception's status and escaped message, `405` with `Allow`, `404`, or a generic 500 (`{"message": "Server Error"}` for JSON clients), and logs the unexpected exception.
