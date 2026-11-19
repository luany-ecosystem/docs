# Configuration

Luany reads settings from two places: environment variables (usually a `.env` file) for values that change between machines, and PHP files in `config/` for application settings.

## Environment variables

A new project ships with `.env.example`; `luany new` and `composer create-project` copy it to `.env`:

```env
APP_NAME=Luany
# development | production — production disables debug pages and live reload.
APP_ENV=development
APP_DEBUG=true
APP_URL=http://localhost:8000
APP_KEY=
APP_TIMEZONE=UTC
APP_LOCALE=en

# sqlite (no server needed) | mysql | pgsql
DB_CONNECTION=sqlite
DB_NAME=database/database.sqlite
```

| Variable | Purpose |
| --- | --- |
| `APP_NAME` | Application name (`config('app.name')`) |
| `APP_ENV` | `development` or `production`. Live reload only runs in `development` (or `local`) |
| `APP_DEBUG` | `true` shows the debug error page and recompiles changed views; always `false` in production |
| `APP_URL` | Base URL used by `url()` and `route()`; its path (sub-folder installs) also prefixes `asset()` |
| `ASSET_URL` | Optional absolute base for `asset()` — a CDN, or full URLs in e-mails |
| `APP_KEY` | Random key written by the installer and `luany key:generate` |
| `APP_TIMEZONE` | Default PHP timezone |
| `APP_LOCALE` | Default locale for [translations](localization.md) |
| `LOG_LEVEL` | Minimum [log](logging.md) level (default `debug`) |
| `DB_CONNECTION`, `DB_NAME`, `DB_HOST`, `DB_PORT`, `DB_USER`, `DB_PASS` | Database connection — see [Database](database.md) |

> **Warning** `.env` contains secrets and is listed in `.gitignore`. Commit `.env.example` instead, with empty or placeholder values.

### How `.env` is loaded

`bootstrap/app.php` loads the file and checks the variables the application cannot run without:

```php
Env::load($ROOT);
Env::required(['APP_ENV', 'APP_URL']);
```

- `Env::load()` does nothing if `.env` does not exist, and only runs once per process.
- Variables already set in the real environment (server config, Docker, CI) **win over `.env`**: the file never overwrites them. In production you can skip `.env` entirely and set real environment variables.
- `Env::required()` throws a `RuntimeException` listing every variable that is missing or empty.

### Reading values

Use the `env()` helper (or `Env::get()`), which checks `$_ENV`, then `$_SERVER`, then `getenv()`:

```php
env('APP_NAME', 'Luany');   // 'Luany' when not set
env('DB_PORT', 3306);
```

A few string values are converted for you:

| Value in `.env` | `env()` returns |
| --- | --- |
| `true`, `(true)` | `true` |
| `false`, `(false)` | `false` |
| `null`, `(null)` | `null` |
| `empty`, `(empty)` | `''` |

Everything else is returned as a string, so cast where needed: `(int) env('DB_PORT', 3306)`.

> **Tip** Call `env()` only from config files and service providers. Elsewhere, read the config value (`config('app.debug')`) so there is a single place that maps environment to settings.

## Configuration files

Every PHP file in `config/` that returns an array becomes a top-level key named after the file. The skeleton ships with `config/app.php`:

```php
return [
    'name'     => env('APP_NAME', 'Luany'),
    'env'      => env('APP_ENV', 'production'),
    'debug'    => (bool) env('APP_DEBUG', false),
    'url'      => env('APP_URL', 'http://localhost:8000'),
    'timezone' => env('APP_TIMEZONE', 'UTC'),

    // Translations live in lang/{locale}.php — use __('key') in code and views.
    'locale'            => env('APP_LOCALE', 'en'),
    'fallback_locale'   => 'en',
    'supported_locales' => ['en'],
];
```

| Key | Used for |
| --- | --- |
| `app.name` | Page titles in the skeleton layout |
| `app.env` | Your own environment checks |
| `app.debug` | Debug mode for LTE's `@dump` / `@dd` directives |
| `app.url` | `url()`, `route()` (falls back to `APP_URL`) |
| `app.asset_url` | `asset()` base (falls back to `ASSET_URL`; empty = root-relative) |
| `app.timezone` | `date_default_timezone_set()` at boot (invalid names are ignored) |
| `app.locale`, `app.fallback_locale`, `app.supported_locales` | The [translator](localization.md) |

The framework also reads these optional files when they exist:

| File | Page |
| --- | --- |
| `config/session.php` | [Session & Flash](session.md#configuration) |
| `config/logging.php` | [Logging](logging.md#configuration) |

Add your own files freely — `config/services.php` is available as `config('services.…')`.

## Accessing configuration

The `config()` helper reads nested values with dot notation:

```php
config('app.name');                  // 'Luany'
config('app.missing', 'default');    // 'default'
config('services.stripe.key');

config(['app.debug' => false]);      // set one or more values at runtime
config()->has('app.timezone');       // true
config()->all();                     // every loaded value
```

Runtime changes last for the current request only; they are not written back to the file.

The repository is the `Luany\Framework\Support\Config` class, bound in the container as `config`, so you can also inject it:

```php
use Luany\Framework\Support\Config;

public function index(Config $config): string
{
    return $config->get('app.name');
}
```

## Debug mode

`APP_DEBUG` is read in more than one place:

- `bootstrap/app.php` passes it to the [error handler](errors.md), which shows the detailed debug page for server errors.
- The view engine recompiles templates when their source changes.
- `config('app.debug')` enables LTE's `@dump` and `@dd`.

> **Warning** Never set `APP_DEBUG=true` on a public server: the debug page shows file paths, the exception message and the full stack trace.
