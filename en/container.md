# Service Container

`Luany\Framework\Application` is the framework's dependency-injection container. It registers services, builds classes with their dependencies through autowiring, injects arguments into controller actions, and implements PSR-11.

## Accessing the container

The application instance is created in `bootstrap/app.php` and is available everywhere through the `app()` helper:

```php
$app = app();                 // the Application
$view = app('view');          // same as app()->make('view')
```

Inside a service provider, the container is passed to `register()` and `boot()` as `$app`. The Application is also registered in itself, so a constructor that type-hints `Luany\Framework\Application` or `Psr\Container\ContainerInterface` receives it.

## Binding services

| Method | Lifetime | Description |
|---|---|---|
| `bind($abstract, $concrete = null)` | Transient | A new instance on every `make()` |
| `singleton($abstract, $concrete = null)` | Shared | Built on the first `make()`, then cached |
| `instance($abstract, $object)` | Shared | Registers an object you already built |

`$abstract` is any string: a class or interface name, or a short key such as `'mailer'`. `$concrete` can be:

- a **factory callable** — receives the Application and an array of parameters;
- a **class name** — built by autowiring (typically to map an interface to an implementation);
- **null** — autowire `$abstract` itself.

```php
use App\Services\Mailer;
use App\Services\SmtpTransport;
use App\Contracts\TransportInterface;
use Luany\Framework\Application;

// Factory
$app->singleton('mailer', fn (Application $app) => new Mailer(
    host: (string) env('MAIL_HOST', 'localhost'),
    port: (int) env('MAIL_PORT', 587),
));

// Interface → implementation
$app->bind(TransportInterface::class, SmtpTransport::class);

// Shared and autowired
$app->singleton(Mailer::class);

// Pre-built object
$app->instance('clock', new \DateTimeImmutable('2026-01-01'));
```

Calling `singleton()` again for the same key replaces the binding and discards a previously built instance.

## Resolving services

```php
$mailer = $app->make('mailer');
$report = $app->make(ReportService::class, ['year' => 2026]);
```

`make()` looks for a registered instance, then a singleton, then a transient binding. If nothing is registered and the identifier is an existing class, the class is built by autowiring. The optional second argument passes **named constructor arguments** that override automatic resolution.

| Exception | When |
|---|---|
| `Luany\Framework\Container\NotFoundException` | Nothing is bound and the class does not exist |
| `Luany\Framework\Container\ContainerException` | The entry exists but cannot be built (abstract class, unresolvable parameter, invalid binding) |
| `Luany\Framework\Container\CircularDependencyException` | The entry depends on itself |

All three extend `ContainerException`, which extends `\RuntimeException` and implements the PSR-11 exception interfaces.

## Autowiring

Unbound concrete classes are built with reflection. Each constructor parameter is resolved in this order:

1. A named argument passed to `make()`.
2. Its class or interface type: `Application` / `ContainerInterface` receive the container; any other type is resolved with `make()`. Interfaces must be bound, otherwise the parameter falls through.
3. Its default value.
4. `null`, when the parameter is nullable.

Otherwise a `ContainerException` names the parameter and class.

```php
namespace App\Services;

use Psr\Log\LoggerInterface;

class ReportService
{
    public function __construct(
        private LoggerInterface $log,   // the framework's logger
        private Mailer $mailer,         // autowired (or the bound singleton)
        private int $year = 2026,       // default value, or make(..., ['year' => 2025])
    ) {}
}
```

```php
$report = app(ReportService::class);
```

Controllers, class-name middleware and global middleware are all created through the container, so the same rules give them constructor injection:

```php
class InvoiceController extends Controller
{
    public function __construct(private ReportService $reports) {}
}
```

### Circular dependencies

If `A` needs `B` and `B` needs `A`, resolution stops with a `CircularDependencyException` showing the full chain instead of recursing forever:

```text
Circular dependency detected while resolving [App\A]: App\A -> App\B -> App\A
```

## Method injection with call()

`call()` invokes a closure, an invokable object, `[object, 'method']` or `[ClassName::class, 'method']` (the class is resolved through the container), resolving each parameter:

1. **By name** — a string key in `$parameters` matching the parameter name.
2. **By type** — a value in `$parameters` that is an instance of the declared class; otherwise the container builds it.
3. **By position** — the remaining `$parameters` values, in order.
4. **Default value**, then `null` for nullable parameters.

```php
$total = $app->call(
    [InvoiceService::class, 'total'],
    ['currency' => 'AOA'],
);

class InvoiceService
{
    public function total(ReportService $reports, string $currency, int $year = 2026): string
    {
        // $reports from the container, $currency by name, $year by default
        return number_format($reports->totalFor($year), 2) . ' ' . $currency;
    }
}
```

A required parameter that none of these rules can fill throws a `ContainerException`.

### Controller actions

The HTTP Kernel calls every route action through `call()`, passing the request and the route parameters by name. That is why all of these signatures work:

```php
Route::get('/posts/{id}', [PostController::class, 'show']);

public function show(Request $request, int $id) { /* ... */ }
public function show(int $id, PostRepository $posts) { /* ... */ }  // order does not matter
public function show($request, $slug) { /* positional: request, then id */ }
```

When a route-model binding resolves to `null` for a parameter that requires an object (for example `show(Post $post)` and no post found), the Kernel responds with **404**. Declare the parameter nullable (`?Post $post`) to handle the missing case yourself. See [Routing](routing.md) and [Controllers](controllers.md).

## PSR-11

The Application implements `Psr\Container\ContainerInterface`:

| Method | Behaviour |
|---|---|
| `get(string $id)` | Same as `make($id)` |
| `has(string $id)` | `true` only for **explicitly registered** entries (`bind`, `singleton`, `instance`) |
| `bound(string $id)` | Alias of `has()` |

`has()` returns `false` for classes that are autowirable but were never registered — `get()` can still build them. Keep that in mind when a PSR-11 consumer checks `has()` before `get()`.

```php
use Psr\Container\ContainerInterface;

function handler(ContainerInterface $container): object
{
    return $container->get(ReportService::class);
}

handler(app());
```

See [PSR Interoperability](psr.md) for using the container with PSR-15 middleware and other PSR-11 containers.

## Framework services

During boot, the HTTP Kernel registers these defaults unless a service provider already bound them, each under a key and a class alias:

| Key | Class alias | Service |
|---|---|---|
| `config` | `Luany\Framework\Support\Config` | Configuration from `config/*.php` |
| `session` | `Luany\Framework\Contracts\SessionInterface` | File session |
| `csrf` | `Luany\Framework\Security\CsrfToken` | CSRF token |
| `view` | `Luany\Lte\Engine` | LTE engine |
| `translator` | `Luany\Framework\Support\Translator` | Translations from `lang/` |
| `log` | `Psr\Log\LoggerInterface` | Daily file logger |

To replace one, bind either the key or the class in a provider's `register()` — see [Service Providers](providers.md).
