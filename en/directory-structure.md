# Directory Structure

A new Luany project is deliberately small: nothing in it is demo code you have to delete first. This page explains what each directory is for.

## Overview

```
my-app/
├── app/
│   ├── Controllers/            HTTP controllers (Controller.php is the base class)
│   ├── Exceptions/Handler.php  Error pages and reporting
│   ├── Http/
│   │   ├── Kernel.php          Global middleware
│   │   └── Middleware/         Application middleware (CsrfMiddleware)
│   ├── Models/                 Database models (User)
│   └── Providers/              Service providers (App, Database)
├── bootstrap/app.php           Creates and configures the application
├── config/app.php              Name, environment, URL, timezone, locale
├── database/
│   ├── migrations/             Schema migrations
│   ├── seeders/                Database seeders
│   └── database.sqlite         SQLite database (created by the installer, git-ignored)
├── public/                     Web root: index.php, .htaccess, assets/, favicon.svg
├── routes/http.php             Route definitions
├── storage/
│   ├── cache/views/            Compiled LTE templates
│   └── logs/                   Log files
├── tests/                      PHPUnit tests
├── views/                      LTE templates
│   ├── components/flash.lte
│   ├── errors/404.lte, 500.lte
│   ├── layouts/main.lte
│   └── pages/home.lte
├── .env / .env.example
├── composer.json
└── package.json                Live-reload dependencies for `luany dev`
```

## The `app` directory

Application code lives here, autoloaded under the `App\` namespace (PSR-4).

| Path | Contents |
| --- | --- |
| `app/Controllers` | `App\Controllers\…` — see [Controllers](controllers.md). `luany make:controller` writes here |
| `app/Http/Kernel.php` | Extends the framework Kernel; lists the [global middleware](middleware.md#global-middleware) |
| `app/Http/Middleware` | Your middleware; `luany make:middleware` writes here |
| `app/Exceptions/Handler.php` | Renders error pages and reports exceptions — see [Error Handling](errors.md) |
| `app/Models` | [Models](models.md) |
| `app/Providers` | [Service providers](providers.md). `DatabaseServiceProvider` opens the connection from `.env` |

Generators create further directories on demand, such as `app/Http/Requests` (`luany make:request`).

## `bootstrap/app.php`

Creates the `Application` container, loads `.env`, registers the service providers and binds the application's Kernel and error Handler. `public/index.php` and the tests both start from this file. See [Request Lifecycle](request-lifecycle.md).

## `config`

PHP files returning arrays, readable with `config('file.key')`. See [Configuration](configuration.md).

## `database`

Migrations, seeders and — for SQLite — the database file. See [Migrations](migrations.md) and [Seeding](seeding.md).

## `public`

The only directory the web server should expose. `index.php` is the front controller for every request; static files go in `public/assets` and are linked with `asset('css/app.css')`.

## `routes`

The Kernel loads `routes/http.php` first, then **every other `*.php` file** in `routes/` in alphabetical order. Splitting routes per area (`routes/admin.php`, `routes/api.php`) needs no registration — `luany make:feature` relies on this. See [Routing](routing.md).

## `storage`

Files written by the framework at runtime. The web server must be able to write here.

| Path | Written by |
| --- | --- |
| `storage/cache/views` | The LTE engine (compiled templates); cleared with `luany cache:clear` |
| `storage/logs` | The [logger](logging.md) — `luany-YYYY-MM-DD.log` |
| `storage/cache/routes.php` | Your deploy script, if you use the [route cache](routing.md#using-the-cache-in-the-kernel) — the Kernel reads it when present |
| `storage/sessions` | [Session](session.md) files — created on first use with `0700` permissions |

Compiled views, logs and session files are git-ignored by the skeleton (`storage/sessions/` since skeleton 1.4.2).

## `views`

LTE templates referenced with dot notation: `view('pages.home')` renders `views/pages/home.lte`. The skeleton's error handler renders `views/errors/404.lte` and `views/errors/500.lte`. See [Template Syntax](lte-syntax.md).

## `lang`

Not created by default. Add `lang/{locale}.php` files when you need [translations](localization.md).

## `tests`

PHPUnit tests under the `Tests\` namespace. The skeleton includes a feature test that boots the real Kernel and requests the home page:

```bash
composer test
```
