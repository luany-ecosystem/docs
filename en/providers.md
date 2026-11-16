# Service Providers

Service providers are where a Luany application is configured: they bind services into the [container](container.md) and run setup code once everything is registered. This page covers the provider lifecycle, registration and common examples.

## Writing a provider

Generate a provider with the CLI:

```bash
luany make:provider Mail
# → app/Providers/MailServiceProvider.php
```

A provider extends `Luany\Framework\ServiceProvider` and overrides one or both methods:

```php
<?php

namespace App\Providers;

use App\Services\Mailer;
use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;

class MailServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton(Mailer::class, fn () => new Mailer(
            host: (string) env('MAIL_HOST', 'localhost'),
            port: (int) env('MAIL_PORT', 587),
        ));
    }

    public function boot(Application $app): void
    {
        //
    }
}
```

Both methods are empty in the base class, so you only implement what you need. You can also implement `Luany\Framework\Contracts\ServiceProviderInterface` directly; it declares the same two methods.

## Lifecycle

| Phase | When | Use it for |
|---|---|---|
| `register(Application $app)` | Immediately, when `$app->register($provider)` is called in `bootstrap/app.php` | Binding services: `bind()`, `singleton()`, `instance()` |
| `boot(Application $app)` | Once, during `Kernel::boot()`, after **all** providers are registered and the framework defaults exist | Using services: `make()`, registering directives, configuring validation, setting static options |

The full boot sequence of the HTTP Kernel is:

1. Register framework defaults (`config`, `session`, `csrf`, `view`, `translator`, `log`) — **only those your providers did not bind**.
2. Wire the container into the router (controllers, middleware, action arguments).
3. Load the route files.
4. Call `boot()` on every provider, in registration order.
5. Configure the `unique` / `exists` validation rules, unless a provider already did.

> **Warning** Do not call `make()` in `register()`. Other providers — and the framework's defaults, such as `view` — are not registered yet. Resolve services in `boot()`, or lazily inside a factory closure.

`boot()` runs once per application; calling `bootProviders()` again has no effect.

## Registering providers

Providers are registered in `bootstrap/app.php`, after the environment is loaded:

```php
// 4. Service Providers — register before kernel boots
$app->register(new App\Providers\AppServiceProvider());
$app->register(new App\Providers\DatabaseServiceProvider());
$app->register(new App\Providers\MailServiceProvider());
```

`register()` is called right away, so order matters only when one provider's `register()` overrides another's binding (the last binding wins). `boot()` methods run in the same order.

Registered providers can be listed with `$app->getProviders()`.

## Overriding framework services

Because the Kernel registers its defaults only for keys that are still unbound, a provider can replace any of them in `register()`. Binding either the key or the class alias is enough — the Kernel aliases the other to yours:

```php
use Luany\Lte\Engine;

public function register(Application $app): void
{
    $app->singleton('view', fn (Application $app) => new Engine(
        viewsPath:  $app->basePath('resources/views'),
        cachePath:  $app->cachePath('views'),
        autoReload: (bool) env('APP_DEBUG', false),
    ));
}
```

The same applies to `config`, `session`, `csrf`, `translator` and `log` (see [Service Container](container.md#framework-services)).

## Example: database connection

The skeleton's `DatabaseServiceProvider` shows the typical split. `register()` only declares factories — no connection is opened until the first query — and `boot()` hands the connection to the models:

```php
<?php

namespace App\Providers;

use Luany\Database\Connection;
use Luany\Database\Model;
use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;
use Luany\Framework\Support\Env;

class DatabaseServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton('connection', function () use ($app) {
            $driver = (string) Env::get('DB_CONNECTION', 'sqlite');
            $name   = (string) Env::get('DB_NAME', $driver === 'sqlite' ? 'database/database.sqlite' : 'luany');

            if ($driver === 'sqlite' && $name !== ':memory:' && !str_starts_with($name, '/') && !preg_match('/^[A-Za-z]:[\\\\\/]/', $name)) {
                $name = $app->basePath($name);
            }

            return Connection::make([
                'driver'   => $driver,
                'host'     => Env::get('DB_HOST', '127.0.0.1'),
                'port'     => Env::get('DB_PORT', $driver === 'pgsql' ? '5432' : '3306'),
                'database' => $name,
                'username' => Env::get('DB_USER', $driver === 'mysql' ? 'root' : null),
                'password' => Env::get('DB_PASS', ''),
            ]);
        });

        // Raw PDO: used by validation rules (unique / exists) and migrations.
        $app->singleton('db', fn (Application $app) => $app->make('connection')->getPdo());
    }

    public function boot(Application $app): void
    {
        Model::setConnection(fn () => $app->make('connection'));
    }
}
```

The `db` binding matters beyond models: the Kernel uses it to power the `unique` and `exists` validation rules. See [Database: Getting Started](database.md).

## Example: custom LTE directives

The view engine is a framework default, so it exists by the time `boot()` runs. Register directives on its compiler:

```php
<?php

namespace App\Providers;

use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;

class ViewServiceProvider extends ServiceProvider
{
    public function boot(Application $app): void
    {
        $app->make('view')->getCompiler()->directive('money', function (?string $args): string {
            return "<?php echo \\Luany\\Lte\\Escaper::escape(number_format((float) ({$args}), 2, '.', ',')); ?>";
        });
    }
}
```

```lte
<p>Total: @money($order->total)</p>
```

Compiled templates are cached per set of registered directives, so adding a directive never reuses stale compiled code. [`luany view:check`](view-check.md) boots the application, so it knows directives registered this way. See [Custom Directives](lte-custom.md).

## Example: validation presence verifier

The `unique` and `exists` rules ask a *presence verifier* how many rows match. If a `db`, `connection` or `PDO` binding exists, the Kernel configures a `DatabasePresenceVerifier` automatically after the providers boot. To use a different connection — or your own lookup — set the verifier in `boot()`; the Kernel leaves an explicitly configured verifier untouched:

```php
use Luany\Framework\Validation\DatabasePresenceVerifier;
use Luany\Framework\Validation\Validator;

public function boot(Application $app): void
{
    Validator::setPresenceVerifier(new DatabasePresenceVerifier(
        fn (): \PDO => $app->make('reporting.connection')->getPdo()
    ));
}
```

`DatabasePresenceVerifier` accepts a `PDO` or a closure returning one (resolved on first use). For a non-SQL source, implement `Luany\Framework\Validation\PresenceVerifierInterface`:

```php
use Luany\Framework\Validation\PresenceVerifierInterface;

final class ApiPresenceVerifier implements PresenceVerifierInterface
{
    public function count(
        string $table,
        string $column,
        array $values,
        ?string $ignoreColumn = null,
        mixed $ignoreValue = null,
    ): int {
        // Return how many distinct $values exist in $table.$column,
        // excluding the row where $ignoreColumn = $ignoreValue.
        return 0;
    }
}
```

See [Validation](validation.md).

## Example: application settings

Static, process-wide options also belong in `boot()` — for example trusting your load balancer:

```php
use Luany\Core\Http\Request;

public function boot(Application $app): void
{
    Request::setTrustedProxies(['10.0.0.0/8']);
}
```
