# Middleware

Middleware wraps the handling of a request: it can inspect or change the request, stop it with its own response, or modify the response on the way out. Luany runs **global middleware** on every request and **route middleware** on matched routes, and also accepts PSR-15 middleware.

## Writing middleware

A middleware implements `Luany\Core\Middleware\MiddlewareInterface`:

```bash
luany make:middleware Auth     # app/Http/Middleware/AuthMiddleware.php
```

```php
namespace App\Http\Middleware;

use Luany\Core\Http\Request;
use Luany\Core\Http\Response;
use Luany\Core\Middleware\MiddlewareInterface;

class AuthMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response
    {
        if (session('user_id') === null) {
            return redirect('/login');          // stop here
        }

        return $next($request);                 // continue
    }
}
```

Work done before `$next()` runs on the way in; work done after it runs on the way out:

```php
public function handle(Request $request, callable $next): Response
{
    $start = microtime(true);

    $response = $next($request);

    return $response->header('X-Response-Time', round((microtime(true) - $start) * 1000) . 'ms');
}
```

Middleware referenced by class name is built through the [container](container.md), so constructor dependencies are injected:

```php
public function __construct(private LoggerInterface $logger) {}
```

## Global middleware

Global middleware runs for every request, before routing — including requests that end in a 404. List it in `app/Http/Kernel.php`, in execution order:

```php
namespace App\Http;

use App\Http\Middleware\CsrfMiddleware;
use Luany\Framework\Http\Kernel as BaseKernel;
use Luany\Framework\Http\Middleware\LiveReloadMiddleware;
use Luany\Framework\Http\Middleware\LocaleMiddleware;

class Kernel extends BaseKernel
{
    protected array $middleware = [
        LiveReloadMiddleware::class,   // first, so it sees the final HTML
        CsrfMiddleware::class,
        LocaleMiddleware::class,
    ];
}
```

The skeleton registers `LiveReloadMiddleware` (active only for local requests with `APP_ENV=development`) and [`CsrfMiddleware`](csrf.md).

## Route middleware

Attach middleware to a single route or to a [group](routing.md#route-groups):

```php
Route::get('/dashboard', [DashboardController::class, 'index'])
    ->middleware(AuthMiddleware::class);

Route::post('/posts', [PostController::class, 'store'])
    ->middleware([AuthMiddleware::class, VerifiedMiddleware::class]);

Route::prefix('admin')->middleware(AuthMiddleware::class)->group(function () {
    Route::get('/users', [AdminController::class, 'users']);
});
```

Group middleware runs before the route's own middleware. Route middleware only runs once a route has matched, after [model binding](routing.md#route-model-binding).

### Middleware with options

To configure middleware per route, pass an instance instead of a class name:

```php
Route::post('/login', [AuthController::class, 'login'])
    ->middleware(new RateLimitMiddleware($limiter, maxAttempts: 5, decaySeconds: 60));
```

Instances cannot be stored in the [route cache](routing.md#route-caching). When that matters, create a small subclass with the options in its constructor and register it by class name (see the [rate limiting](#rate-limiting) example).

## PSR-15 middleware

Route middleware may also implement `Psr\Http\Server\MiddlewareInterface`, given as an instance or a class name. Luany converts the request to PSR-7 and back automatically:

```php
Route::get('/api/data', [ApiController::class, 'data'])
    ->middleware(new \Middlewares\Cors($analyzer));
```

The bridge needs a PSR-7 implementation: `nyholm/psr7` and `guzzlehttp/psr7` are detected automatically. See [PSR Interoperability](psr.md).

The Kernel's global list only accepts Luany middleware. Wrap a PSR-15 middleware in `Luany\Core\Psr\Psr15Middleware` to use it globally:

```php
use Luany\Core\Psr\Psr15Middleware;
use Luany\Framework\Application;

class Kernel extends BaseKernel
{
    public function __construct(Application $app)
    {
        parent::__construct($app);

        $this->middleware[] = new Psr15Middleware(new \Middlewares\ResponseTime());
    }
}
```

## CORS

`Luany\Core\Middleware\CorsMiddleware` answers preflight `OPTIONS` requests with `204 No Content` and adds CORS headers to every other response.

Register it **globally**: a preflight request does not match any route, so route middleware would never see it.

```php
protected array $middleware = [
    \Luany\Core\Middleware\CorsMiddleware::class,
    CsrfMiddleware::class,
];
```

Without arguments it allows every origin, without credentials. To configure it, bind it in a provider's `register()` method — the Kernel builds global middleware through the container:

```php
use Luany\Core\Middleware\CorsMiddleware;

$app->bind(CorsMiddleware::class, fn () => new CorsMiddleware(
    allowedOrigins:   ['https://app.example.com', 'https://*.example.com'],
    allowCredentials: true,
    exposedHeaders:   ['X-Total-Count'],
));
```

| Option | Default |
| --- | --- |
| `allowedOrigins` | `['*']` |
| `allowedMethods` | `['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS']` |
| `allowedHeaders` | `['Content-Type', 'Authorization', 'X-Requested-With']` |
| `exposedHeaders` | `[]` |
| `allowCredentials` | `false` |
| `maxAge` | `86400` (seconds a preflight may be cached) |

Origins may be exact (`https://app.example.com`) or wildcard subdomains: `https://*.example.com` (HTTPS only) or `*.example.com` (any scheme). A matching origin is echoed back with `Vary: Origin`; an origin that is not allowed gets no CORS headers.

> **Warning** `allowCredentials: true` combined with the `*` origin throws an `InvalidArgumentException` — it would let any website make authenticated requests. List the origins explicitly.

## Rate limiting

`Luany\Core\Middleware\RateLimitMiddleware` limits requests per client IP over a time window, using a `RateLimiterInterface` store:

| Store | Use |
| --- | --- |
| `Luany\Core\RateLimit\FileRateLimiter` | One JSON file per key under a directory; single-server deployments |
| `Luany\Core\RateLimit\InMemoryRateLimiter` | Lives for one PHP process only — tests and long-running workers |

```php
use Luany\Core\Middleware\RateLimitMiddleware;
use Luany\Core\RateLimit\FileRateLimiter;

$limiter = new FileRateLimiter(base_path('storage/rate-limits'));

Route::post('/login', [AuthController::class, 'login'])
    ->middleware(new RateLimitMiddleware($limiter, maxAttempts: 5, decaySeconds: 60, keyPrefix: 'login'));
```

| Argument | Default | Meaning |
| --- | --- | --- |
| `limiter` | — | The store |
| `maxAttempts` | `60` | Requests allowed per window |
| `decaySeconds` | `60` | Window length |
| `keyPrefix` | `'rl'` | Separates counters between routes |

Allowed responses get `X-RateLimit-Limit` and `X-RateLimit-Remaining` headers. Over the limit, the middleware returns `429 Too Many Requests` with `Retry-After`.

The counter key is `{prefix}:{ip}`. Override `keyFor()` to limit per user or API token, and set the options in the constructor so the middleware can be registered by class name:

```php
namespace App\Http\Middleware;

use Luany\Core\Http\Request;
use Luany\Core\Middleware\RateLimitMiddleware;
use Luany\Core\RateLimit\FileRateLimiter;

class ApiThrottle extends RateLimitMiddleware
{
    public function __construct()
    {
        parent::__construct(new FileRateLimiter(base_path('storage/rate-limits')), 120, 60, 'api');
    }

    protected function keyFor(Request $request): string
    {
        return 'api:' . ($request->header('Authorization') ?? $request->ip());
    }
}
```

## Trusted proxies

Behind a load balancer or reverse proxy, `REMOTE_ADDR` is the proxy's address. Tell the request which proxies to trust so `$request->ip()` and `$request->isSecure()` read `X-Forwarded-For` and `X-Forwarded-Proto` — for example in `bootstrap/app.php` or a provider's `boot()` method:

```php
use Luany\Core\Http\Request;

Request::setTrustedProxies(['10.0.0.0/8', '127.0.0.1', '::1']);
```

IPs and CIDR ranges (IPv4 and IPv6) are accepted. Forwarded headers from any other address are ignored, so clients cannot spoof their IP — which matters for rate limiting.

> **Note** The session cookie's automatic `secure` flag does not use these headers. Behind a TLS-terminating proxy, set `'secure' => true` in [`config/session.php`](session.md#configuration).
