# Request Lifecycle

This page follows a request from `public/index.php` to the response sent to the browser. Knowing the order of these steps makes it clear where to register services, middleware and routes.

## Overview

```
public/index.php
  └─ bootstrap/app.php     autoload → Application → .env → providers register()
  └─ Kernel::boot()        default services → routes → providers boot()
  └─ Kernel::handle()      global middleware → router → route middleware → action
  └─ Response::send()
  └─ Kernel::terminate()
```

## 1. The front controller

Every request enters through `public/index.php`:

```php
$app = require dirname(__DIR__) . '/bootstrap/app.php';

$kernel   = $app->make(Kernel::class);
$kernel->boot();

$request  = Request::fromGlobals();
$response = $kernel->handle($request);
$response->send();

$kernel->terminate($request, $response);
```

## 2. Bootstrapping

`bootstrap/app.php` creates the application and returns it:

1. Loads Composer's autoloader.
2. Creates `Luany\Framework\Application` — the [service container](container.md). It registers itself, so `app()` works from here on.
3. Loads `.env` with `Env::load()` and asserts `APP_ENV` and `APP_URL` with `Env::required()`.
4. Registers the service providers. `$app->register()` calls each provider's `register()` method **immediately**; `boot()` is deferred.
5. Binds `App\Http\Kernel` as the HTTP Kernel and `App\Exceptions\Handler` as the error handler.

## 3. Booting the Kernel

`Kernel::boot()` runs once, in this order:

| Step | What happens |
| --- | --- |
| 1. `config` | Loads every file in `config/` |
| 2. `session` | Registers the file session (started lazily, on first use) |
| 3. `csrf` | Registers the CSRF token service |
| 4. `view` | Registers the LTE engine, sets LTE's debug flag, applies `app.timezone`, connects `Route::view()` |
| 5. `translator` | Registers the translator from `app.locale` and friends |
| 6. `log` | Registers the PSR-3 file logger |
| 7. Resolver | Makes the router build controllers and middleware through the container and call actions with [method injection](controllers.md#method-injection) |
| 8. Routes | Loads the [route cache](routing.md#using-the-cache-in-the-kernel) `storage/cache/routes.php` when it exists; otherwise requires `routes/http.php`, then every other `routes/*.php` alphabetically |
| 9. Providers | Calls `boot()` on every registered provider |
| 10. Validation | Connects the `unique` / `exists` rules to the `db` binding, if one exists |

### Default services

Steps 1–6 only register a service when no provider already did. Each service is available under a key and a class name:

| Key | Class / interface | Default |
| --- | --- | --- |
| `config` | `Luany\Framework\Support\Config` | Files in `config/` |
| `session` | `Luany\Framework\Contracts\SessionInterface` | `FileSession` in `storage/sessions` |
| `csrf` | `Luany\Framework\Security\CsrfToken` | Token stored in the session |
| `view` | `Luany\Lte\Engine` | `views/`, cache in `storage/cache/views` |
| `translator` | `Luany\Framework\Support\Translator` | `lang/{locale}.php` |
| `log` | `Psr\Log\LoggerInterface` | `FileLogger` in `storage/logs` |

To replace one, bind either the key or the class in a provider's `register()` method; the Kernel aliases the other name to your binding:

```php
public function register(Application $app): void
{
    $app->singleton('log', fn () => new \Monolog\Logger('app'));
}
```

Because routes are loaded **before** providers boot, a provider's `boot()` method can inspect or add routes, and every service a route file needs at load time must be bound in `register()`.

## 4. Handling the request

`Request::fromGlobals()` builds the [request](requests.md) from PHP's globals: it applies `_method` form spoofing, strips the sub-directory base path and decodes JSON bodies.

`Kernel::handle()` then:

1. Sends the request through the **global middleware** listed in `App\Http\Kernel::$middleware` (in the skeleton: live reload, then CSRF). Global middleware runs for every request, including ones that match no route.
2. Passes it to the **router**, which finds the first route whose URI and method match. A matching URI with the wrong method throws a 405; no match throws a 404.
3. Resolves route parameters (including [model bindings](routing.md#route-model-binding)) and converts those declared `int` / `float` in the action — an invalid number is a 404.
4. Runs the **route middleware** (group middleware first, then the route's own).
5. Calls the **action** — a closure or controller method — resolving its arguments through the container.
6. Converts the return value into a `Response`: strings become HTML, arrays and objects become JSON.

## 5. Exceptions

Any exception thrown during steps 1–6 is turned into a response by the Kernel:

| Exception | Response |
| --- | --- |
| `ValidationException` | 422 JSON for JSON clients, otherwise a redirect back with errors flashed |
| HTTP exceptions (`abort()`, 404, 405, `findOrFail()`) for JSON clients | JSON `{"message": "…"}` with the right status |
| Anything else | Reported and rendered by `App\Exceptions\Handler` — `renderJson()` for JSON clients, `render()` otherwise |

See [Error Handling](errors.md).

## 6. Sending and terminating

`Response::send()` writes the status line, headers, cookies and body. Finally `Kernel::terminate()` is called — an empty hook you can override in `App\Http\Kernel` for work after the response is sent:

```php
public function terminate(Request $request, Response $response): void
{
    logger('Handled {method} {uri}', ['method' => $request->method(), 'uri' => $request->uri()]);
}
```

## Handling requests in tests

The same steps work without a web server — build a `Request` by hand and pass it to the Kernel:

```php
$app    = require dirname(__DIR__, 2) . '/bootstrap/app.php';
$kernel = $app->make(\Luany\Framework\Http\Kernel::class);
$kernel->boot();

$response = $kernel->handle(new Request('GET', '/', server: ['REMOTE_ADDR' => '127.0.0.1']));

$this->assertSame(200, $response->getStatusCode());
```
