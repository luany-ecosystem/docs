# Error Handling

Every exception thrown while handling a request is caught by the Kernel and turned into a response. Expected HTTP outcomes (404, 403, 422…) get the right status; unexpected errors are logged and shown as a 500 page — or as a detailed debug page in development.

## How exceptions become responses

| Exception | JSON client | Browser |
| --- | --- | --- |
| `ValidationException` | 422 `{"message", "errors"}` | Redirect back, errors flashed — see [Validation](validation.md) |
| `HttpException` (`abort()`) | `{"message": "…"}` with its status and headers | Rendered by the Handler |
| No route matched | 404 `{"message": "Not Found"}` | Rendered by the Handler |
| Wrong method | 405 with `Allow` header | Rendered by the Handler |
| Anything else | Reported and rendered by the Handler | Reported and rendered by the Handler |

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

For each exception the Kernel calls `report()`, then `render()`.

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
| `statusCodeFor($e)` | `HttpException` status, 422 for validation, 404 / 405 for routing, otherwise 500 |
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

> **Note** The skeleton's Handler declares `protected array $dontReport = [];`, which **replaces** that list — so 404s and `abort()` calls are logged as errors. Copy the list above into your Handler to keep expected HTTP errors out of the log, and append your own classes.

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

HTTP errors and validation errors are already JSON for JSON clients. Other exceptions go through `Handler::render()`, which has no access to the request; if your API needs a JSON 500, check the `Accept` header there:

```php
public function render(\Throwable $e): Response
{
    if (str_contains($_SERVER['HTTP_ACCEPT'] ?? '', 'application/json')) {
        return Response::json(['message' => $this->debug ? $e->getMessage() : 'Server Error'], $this->statusCodeFor($e));
    }

    return parent::render($e);
}
```

## When the handler itself fails

If no Handler is bound, or `render()` throws, the Kernel falls back to plain responses: the HTTP exception's status and escaped message, `405` with `Allow`, `404`, or a generic 500 (`{"message": "Server Error"}` for JSON clients), and logs the unexpected exception.
