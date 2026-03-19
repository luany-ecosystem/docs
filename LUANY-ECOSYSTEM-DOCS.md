# Luany Ecosystem — Comprehensive Documentation

> **Luany** — AST-compiled PHP MVC framework with an explicit request lifecycle and zero-regex template engine.

**Version**: 0.3.x &nbsp;|&nbsp; **PHP**: ≥ 8.1 &nbsp;|&nbsp; **License**: MIT
**Author**: António Ambrósio Ngola &nbsp;|&nbsp; **Org**: [luany-ecosystem](https://github.com/luany-ecosystem)

---

## Table of Contents

1. [Package Overview](#1-package-overview)
2. [Installation](#2-installation)
3. [Architecture](#3-architecture)
4. [luany/core — HTTP & Routing](#4-luanycore--http--routing)
5. [luany/database — ORM & Migrations](#5-luanydatabase--orm--migrations)
6. [luany/framework — Application Layer](#6-luanyframework--application-layer)
7. [luany/lte — Template Engine](#7-luanylte--template-engine)
8. [luany/cli — Command-Line Tool](#8-luanycli--command-line-tool)
9. [luany/luany — Application Skeleton](#9-luanyluany--application-skeleton)
10. [Configuration Reference](#10-configuration-reference)
11. [Environment Variables](#11-environment-variables)
12. [CLI Command Reference](#12-cli-command-reference)
13. [LTE Template Reference](#13-lte-template-reference)
14. [Integration Guide](#14-integration-guide)
15. [Upgrade Guide](#15-upgrade-guide)

---

## 1. Package Overview

| Package | Composer Name | Description | Version |
|---------|---------------|-------------|---------|
| **Core** | `luany/core` | HTTP Request/Response, Router, Middleware Pipeline | v0.2.4 |
| **Database** | `luany/database` | PDO Connection, Query Builder, Model, Migrations | v0.1.3 |
| **Framework** | `luany/framework` | Application container, Kernel, Service Providers, i18n | v0.3.1 |
| **LTE** | `luany/lte` | AST-compiled template engine (`.lte` files) | v0.2.x |
| **CLI** | `luany/cli` | `luany` command-line tool, scaffolding, migrations | v0.2.2 |
| **Skeleton** | `luany/luany` | Ready-to-use application template | — |

### Dependency Graph

```
luany/luany (skeleton)
├── luany/framework
│   ├── luany/core
│   └── luany/lte
├── luany/database
└── luany/cli (dev tool)
```

---

## 2. Installation

### New Project

```bash
composer create-project luany/luany my-app
cd my-app
luany key:generate
luany serve
```

### Existing Project

```bash
composer require luany/framework luany/database
composer require --dev luany/cli
```

### Global CLI

```bash
composer global require luany/cli
luany new my-app
```

---

## 3. Architecture

### Request Lifecycle

```
Browser → public/index.php
           ↓
       bootstrap/app.php
       (Env::load, Providers, Singleton bindings)
           ↓
       Kernel::boot()
       (Register routes, global middleware)
           ↓
       Kernel::handle(Request)
           ↓
       Pipeline → Middleware → Router → Controller → Response
           ↓
       Response::send()
           ↓
       Kernel::terminate()
```

### Directory Structure (Skeleton)

```
my-app/
├── app/
│   ├── Controllers/       # HTTP controllers
│   ├── Exceptions/        # Custom Handler
│   ├── Http/
│   │   ├── Kernel.php     # HTTP Kernel
│   │   └── Middleware/     # Application middleware
│   ├── Models/            # Eloquent-style models
│   ├── Providers/         # Service providers
│   └── Support/           # Helpers
├── bootstrap/
│   └── app.php            # Application bootstrap
├── config/
│   ├── app.php            # Application config
│   └── mail.php           # Mail config
├── database/
│   └── migrations/        # Timestamped migrations
├── lang/                  # Translation files (en.php, pt.php)
├── public/
│   ├── index.php          # Front controller
│   └── assets/            # CSS, JS, images
├── routes/
│   └── http.php           # Route definitions
├── storage/
│   ├── cache/views/       # Compiled LTE cache
│   └── logs/              # Log files
├── views/
│   ├── components/        # Reusable partials
│   ├── layouts/           # Layout templates
│   └── pages/             # Page views
├── .env                   # Environment config
└── composer.json
```

---

## 4. luany/core — HTTP & Routing

**Namespace**: `Luany\Core`

### 4.1 Request

**Class**: `Luany\Core\Http\Request`

Immutable HTTP request wrapper. Created via `Request::fromGlobals()` or manual construction.

```php
// From superglobals (index.php)
$request = Request::fromGlobals();

// Manual construction (testing)
$request = new Request(
    method: 'GET',
    uri: '/users/42',
    headers: ['Accept' => 'application/json'],
    body: '',
    queryParams: [],
    postParams: [],
    cookies: [],
    serverParams: $_SERVER
);
```

**Public Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `fromGlobals` | `static fromGlobals(): self` | Create from PHP superglobals |
| `method` | `method(): string` | HTTP method (GET, POST, etc.) |
| `uri` | `uri(): string` | Request URI path |
| `header` | `header(string $name, ?string $default = null): ?string` | Get a header (case-insensitive) |
| `body` | `body(): string` | Raw request body |
| `all` | `all(): array` | Merged query + post params |
| `input` | `input(string $key, mixed $default = null): mixed` | Get a single input value |
| `query` | `query(string $key, mixed $default = null): mixed` | GET parameter |
| `post` | `post(string $key, mixed $default = null): mixed` | POST parameter |
| `cookie` | `cookie(string $name, ?string $default = null): ?string` | Cookie value |
| `server` | `server(string $key, mixed $default = null): mixed` | Server param |
| `only` | `only(array $keys): array` | Subset of inputs |
| `has` | `has(string $key): bool` | Check if key exists |
| `isMethod` | `isMethod(string $method): bool` | Check HTTP method |
| `isGet` / `isPost` | `isGet(): bool` / `isPost(): bool` | Method shortcuts |
| `routeParams` | `routeParams(): array` | Parameters extracted by router |
| `withRouteParams` | `withRouteParams(array $params): self` | Clone with route params |

### 4.2 Response

**Class**: `Luany\Core\Http\Response`

HTTP response builder with static factory methods.

```php
// Basic
return Response::make('<h1>Hello</h1>', 200);

// JSON
return Response::json(['user' => $user]);

// Redirect
return Response::redirect('/dashboard');

// Error responses
return Response::notFound();
return Response::serverError();
```

**Public Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `make` | `static make(string $body = '', int $status = 200): self` | Create response |
| `json` | `static json(mixed $data, int $status = 200): self` | JSON response |
| `redirect` | `static redirect(string $url, int $status = 302): self` | Redirect |
| `notFound` | `static notFound(): self` | 404 response |
| `serverError` | `static serverError(): self` | 500 response |
| `header` | `header(string $name, string $value): self` | Add header |
| `withCookie` | `withCookie(string $name, string $value, array $options = []): self` | Set cookie |
| `getStatusCode` | `getStatusCode(): int` | Get status code |
| `getBody` | `getBody(): string` | Get body content |
| `getHeaders` | `getHeaders(): array` | Get all headers |
| `send` | `send(): void` | Send to client (headers + body) |

### 4.3 Router

**Class**: `Luany\Core\Routing\Router`

Collects route definitions and dispatches requests.

```php
use Luany\Core\Routing\Route;

Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
Route::get('/users/{id}', [UserController::class, 'show']);
Route::put('/users/{id}', [UserController::class, 'update']);
Route::delete('/users/{id}', [UserController::class, 'destroy']);

// Resource routes (all 7 CRUD routes)
Route::resource('products', ProductController::class);

// Route groups
Route::group(['prefix' => '/admin', 'middleware' => [AuthMiddleware::class]], function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
});
```

**Route Static Methods** (on `Luany\Core\Routing\Route`):

| Method | Signature |
|--------|-----------|
| `get` | `static get(string $uri, array\|callable $action): void` |
| `post` | `static post(string $uri, array\|callable $action): void` |
| `put` | `static put(string $uri, array\|callable $action): void` |
| `patch` | `static patch(string $uri, array\|callable $action): void` |
| `delete` | `static delete(string $uri, array\|callable $action): void` |
| `resource` | `static resource(string $name, string $controller): void` |
| `group` | `static group(array $attributes, callable $callback): void` |

**Router Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `handle` | `handle(Request $request): Response` | Dispatch request to matching route |
| `addRoute` | `addRoute(string $method, string $uri, array\|callable $action): void` | Register a route |

**Route Parameters**: Parameters in `{braces}` are extracted and injected into controller methods.

```php
// routes/http.php
Route::get('/users/{id}', [UserController::class, 'show']);

// Controller
class UserController {
    public function show(Request $request, string $id): string {
        // $id = '42' from /users/42
    }
}
```

### 4.4 MiddlewareInterface

**Interface**: `Luany\Core\Middleware\MiddlewareInterface`

```php
interface MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response;
}
```

### 4.5 Pipeline

**Class**: `Luany\Core\Middleware\Pipeline`

Executes middleware in sequence, wrapping the final handler.

```php
$pipeline = new Pipeline();
$response = $pipeline
    ->send($request)
    ->through([AuthMiddleware::class, LogMiddleware::class])
    ->then(function (Request $request) {
        return $router->handle($request);
    });
```

**Methods**:

| Method | Signature |
|--------|-----------|
| `send` | `send(Request $request): self` |
| `through` | `through(array $middleware): self` |
| `then` | `then(callable $destination): Response` |

### 4.6 RouteNotFoundException

**Class**: `Luany\Core\Routing\RouteNotFoundException`

Thrown when no route matches the request. Extends `\RuntimeException`.

---

## 5. luany/database — ORM & Migrations

**Namespace**: `Luany\Database`

### 5.1 Connection

**Class**: `Luany\Database\Connection`

PDO wrapper with singleton pattern.

```php
Connection::configure([
    'host'    => '127.0.0.1',
    'port'    => 3306,
    'dbname'  => 'my_app',
    'user'    => 'root',
    'pass'    => '',
    'charset' => 'utf8mb4',
]);

$pdo = Connection::getInstance();
```

**Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `configure` | `static configure(array $config): void` | Set connection config |
| `getInstance` | `static getInstance(): \PDO` | Get/create PDO singleton |
| `disconnect` | `static disconnect(): void` | Close connection |

### 5.2 QueryBuilder

**Class**: `Luany\Database\QueryBuilder`

Fluent query builder.

```php
$builder = new QueryBuilder(Connection::getInstance());

// SELECT
$users = $builder->table('users')
    ->select('id', 'name', 'email')
    ->where('active', '=', 1)
    ->orderBy('name', 'ASC')
    ->limit(10)
    ->get();

// INSERT
$builder->table('users')->insert([
    'name'  => 'João',
    'email' => 'joao@example.com',
]);

// UPDATE
$builder->table('users')
    ->where('id', '=', 42)
    ->update(['name' => 'João Silva']);

// DELETE
$builder->table('users')
    ->where('id', '=', 42)
    ->delete();
```

**Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `table` | `table(string $table): self` | Set target table |
| `select` | `select(string ...$columns): self` | Columns to select |
| `where` | `where(string $column, string $operator, mixed $value): self` | WHERE clause |
| `orderBy` | `orderBy(string $column, string $direction = 'ASC'): self` | ORDER BY |
| `limit` | `limit(int $limit): self` | LIMIT |
| `offset` | `offset(int $offset): self` | OFFSET |
| `get` | `get(): array` | Execute SELECT, return rows |
| `first` | `first(): ?array` | First row or null |
| `insert` | `insert(array $data): bool` | Insert row |
| `update` | `update(array $data): int` | Update rows, return count |
| `delete` | `delete(): int` | Delete rows, return count |
| `count` | `count(): int` | Count matching rows |
| `raw` | `raw(string $sql, array $bindings = []): Result` | Raw query |

### 5.3 Result

**Class**: `Luany\Database\Result`

Wraps PDOStatement for convenient result access. Implements `Countable` and `IteratorAggregate`.

| Method | Signature |
|--------|-----------|
| `all` | `all(): array` |
| `first` | `first(): ?array` |
| `count` | `count(): int` |
| `column` | `column(int $index = 0): array` |
| `lastInsertId` | `lastInsertId(): string` |

### 5.4 Model

**Class**: `Luany\Database\Model` (abstract)

Active Record ORM base class.

```php
namespace App\Models;

use Luany\Database\Model;

class User extends Model
{
    protected string $table      = 'users';
    protected string $primaryKey = 'id';
    protected array  $fillable   = ['name', 'email', 'password'];
    protected array  $hidden     = ['password'];
}
```

**Usage**:

```php
// Find by ID
$user = User::find(42);

// All records (with optional ordering)
$users = User::all('created_at DESC');

// Create
$user = User::create([
    'name'  => 'Maria',
    'email' => 'maria@example.com',
]);

// Update
$user->update(['name' => 'Maria Silva']);

// Delete
$user->delete();

// Access attributes
echo $user->name;
echo $user->email;

// Serialization (hides $hidden fields)
$array = $user->toArray();
$json  = $user->toJson();
```

**Model Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `find` | `static find(mixed $id): ?static` | Find by primary key |
| `all` | `static all(string $orderBy = ''): array` | All records (validated order) |
| `create` | `static create(array $attributes): static` | Insert and return instance |
| `update` | `update(array $attributes): bool` | Update this record |
| `delete` | `delete(): bool` | Delete this record |
| `toArray` | `toArray(): array` | Convert to array (excludes hidden) |
| `toJson` | `toJson(): string` | Convert to JSON |
| `fill` | `fill(array $attributes): void` | Mass-assign fillable attributes |

### 5.5 Migration System

#### Migration (abstract)

**Class**: `Luany\Database\Migration\Migration`

```php
use Luany\Database\Migration\Migration;

class CreateUsersTable extends Migration
{
    public function up(\PDO $pdo): void
    {
        $pdo->exec("
            CREATE TABLE IF NOT EXISTS `users` (
                `id` INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                `name` VARCHAR(255) NOT NULL,
                `email` VARCHAR(150) NOT NULL UNIQUE,
                `created_at` TIMESTAMP DEFAULT CURRENT_TIMESTAMP
            ) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4
        ");
    }

    public function down(\PDO $pdo): void
    {
        $pdo->exec("DROP TABLE IF EXISTS `users`");
    }
}
```

#### MigrationRepository

**Class**: `Luany\Database\Migration\MigrationRepository`

Manages the `_migrations` tracking table. Creates it automatically on first use.

| Method | Description |
|--------|-------------|
| `ensureTable(): void` | Create `_migrations` table if missing |
| `getRan(): array` | List of already-run migration names |
| `log(name, batch): void` | Record a migration as run |
| `removeLast(): array` | Get + remove last batch |
| `status(): array` | All migrations with ran/batch status |

#### MigrationRunner

**Class**: `Luany\Database\Migration\MigrationRunner`

Orchestrates running, rolling back, and inspecting migrations.

| Method | Signature | Description |
|--------|-----------|-------------|
| `run` | `run(?callable $callback = null): int` | Run pending migrations |
| `rollback` | `rollback(?callable $callback = null): int` | Rollback last batch |
| `status` | `status(): array` | Get all migration statuses |
| `dropAll` | `dropAll(\PDO $pdo): void` | Drop all tables |

---

## 6. luany/framework — Application Layer

**Namespace**: `Luany\Framework`

### 6.1 Application

**Class**: `Luany\Framework\Application`

IoC container and application kernel. Implements `ApplicationInterface`.

```php
$app = new Application('/path/to/project');

// Bind a singleton
$app->singleton('db', fn($app) => new Connection($config));

// Bind a factory (new instance each call)
$app->bind('logger', fn($app) => new Logger());

// Resolve
$db = $app->make('db');

// Register provider
$app->register(new AppServiceProvider());
```

**Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `__construct` | `__construct(string $basePath)` | Create application |
| `basePath` | `basePath(string $path = ''): string` | Absolute path to project root |
| `bind` | `bind(string $abstract, callable $factory): void` | Register factory binding |
| `singleton` | `singleton(string $abstract, callable $factory): void` | Register singleton |
| `make` | `make(string $abstract): mixed` | Resolve binding |
| `has` | `has(string $abstract): bool` | Check if binding exists |
| `register` | `register(ServiceProviderInterface $provider): void` | Register service provider |
| `boot` | `boot(): void` | Boot all registered providers |
| `getInstance` | `static getInstance(): self` | Get singleton instance |

### 6.2 Contracts (Interfaces)

#### ApplicationInterface

```php
interface ApplicationInterface
{
    public function basePath(string $path = ''): string;
    public function bind(string $abstract, callable $factory): void;
    public function singleton(string $abstract, callable $factory): void;
    public function make(string $abstract): mixed;
    public function has(string $abstract): bool;
    public function register(ServiceProviderInterface $provider): void;
    public function boot(): void;
}
```

#### ServiceProviderInterface

```php
interface ServiceProviderInterface
{
    public function register(Application $app): void;
    public function boot(Application $app): void;
}
```

#### KernelInterface

```php
interface KernelInterface
{
    public function boot(): void;
    public function handle(Request $request): Response;
    public function terminate(Request $request, Response $response): void;
}
```

### 6.3 ServiceProvider

**Class**: `Luany\Framework\ServiceProvider` (abstract)

Convenience base class with empty default implementations.

```php
class MailServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton('mailer', fn() => new Mailer(
            host: env('MAIL_HOST'),
            port: env('MAIL_PORT', 587),
        ));
    }

    public function boot(Application $app): void
    {
        // Post-registration setup
    }
}
```

### 6.4 HTTP Kernel

**Class**: `Luany\Framework\Http\Kernel`

Implements `KernelInterface`. Manages the request lifecycle with middleware pipeline.

```php
// In app/Http/Kernel.php
class Kernel extends \Luany\Framework\Http\Kernel
{
    protected array $middleware = [
        LocaleMiddleware::class,
        CsrfMiddleware::class,
    ];

    protected function routes(): void
    {
        require base_path('routes/http.php');
    }
}
```

**Error handling**: Uses a hybrid try/catch — inner catch for route exceptions (preserving middleware "after" phase), outer catch for middleware exceptions. Both flow through the exception Handler.

### 6.5 Exception Handler

**Class**: `Luany\Framework\Exceptions\Handler` (abstract)

```php
// In app/Exceptions/Handler.php
class Handler extends \Luany\Framework\Exceptions\Handler
{
    public function report(\Throwable $e): void
    {
        // Send to Sentry, log, etc.
        parent::report($e);
    }

    public function render(\Throwable $e): Response
    {
        if ($e instanceof ModelNotFoundException) {
            return Response::notFound();
        }
        return parent::render($e);
    }
}
```

**Methods**:

| Method | Description |
|--------|-------------|
| `__construct(bool $debug = false)` | Enable/disable debug mode |
| `report(\Throwable $e): void` | Log the exception (default: `error_log`) |
| `render(\Throwable $e): Response` | Render exception to HTTP response |

In **debug mode**, renders a detailed HTML error page with class, message, file, line, trace, URI, and method.

### 6.6 LocaleMiddleware

**Class**: `Luany\Framework\Http\Middleware\LocaleMiddleware`

Detects locale on every request and sets it on the Translator.

**Detection order** (highest to lowest):
1. Cookie `app_locale` — explicit user preference
2. `Accept-Language` HTTP header — browser preference
3. `APP_LOCALE` env variable — application default
4. `'en'` — hardcoded fallback

### 6.7 Env

**Class**: `Luany\Framework\Support\Env`

Wrapper around `vlucas/phpdotenv`. Casts string values to PHP types.

```php
Env::load('/path/to/project');

$host  = Env::get('DB_HOST', 'localhost');
$debug = Env::get('APP_DEBUG', false); // returns boolean true/false

Env::required(['DB_HOST', 'DB_NAME']); // throws RuntimeException if missing
```

**Type casting**: `'true'` → `true`, `'false'` → `false`, `'null'` → `null`, `'empty'` → `''`.

### 6.8 Translator

**Class**: `Luany\Framework\Support\Translator`

Lightweight i18n engine. Translation files live in `lang/{locale}.php`.

```php
$translator = new Translator(
    langPath: '/path/to/lang',
    locale: 'en',
    fallback: 'en',
    supported: ['en', 'pt'],
);

$translator->get('nav.home');
$translator->get('footer.copyright', ['year' => '2026']);
$translator->setLocale('pt');
$translator->getLocale();    // 'pt'
$translator->getSupported(); // ['en', 'pt']
```

**Translation file** (`lang/en.php`):

```php
return [
    'nav.home'         => 'Home',
    'footer.copyright' => '© :year Luany',
];
```

### 6.9 Global Helper Functions

Defined in `Luany\Framework\Support\helpers.php`, auto-loaded:

| Function | Signature | Description |
|----------|-----------|-------------|
| `app()` | `app(?string $abstract = null): mixed` | Application instance or resolve binding |
| `env()` | `env(string $key, mixed $default = null): mixed` | Environment variable |
| `base_path()` | `base_path(string $path = ''): string` | Absolute path to project root |
| `view()` | `view(string $name, array $data = []): string` | Render LTE view |
| `redirect()` | `redirect(string $url, int $status = 302): Response` | Redirect response |
| `response()` | `response(string $body = '', int $status = 200): Response` | Create response |
| `__()` | `__(string $key, array $replace = []): string` | Translate key |
| `locale()` | `locale(): string` | Current locale code |

---

## 7. luany/lte — Template Engine

**Namespace**: `Luany\Lte`

LTE (Luany Template Engine) compiles `.lte` templates into cached PHP files via AST parsing.

### 7.1 Engine

**Class**: `Luany\Lte\Engine`

Orchestrates the compile → cache → evaluate pipeline.

```php
$engine = new Engine(
    viewsPath: '/path/to/views',
    cachePath: '/path/to/cache',  // default: sys_get_temp_dir()/lte_cache
    autoReload: true,              // always recompile (dev mode)
);

$html = $engine->render('pages.home', ['title' => 'Welcome']);
```

**Render flow**:
1. `SectionStack::reset()` + `AssetStack::reset()`
2. Compile + evaluate child view (populates sections/assets, sets layout)
3. If `@extends` was used → compile + evaluate layout (consumes sections)
4. Return final HTML

**Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `render` | `render(string $view, array $data = []): string` | Render view to HTML |
| `getCompiler` | `getCompiler(): Compiler` | Access compiler (custom directives) |
| `clearCache` | `clearCache(): void` | Delete all cached files |
| `findView` | `findView(string $view): ?string` | Resolve dot-name to file path |
| `setAutoReload` | `setAutoReload(bool $flag): void` | Toggle auto-recompile |

### 7.2 Parser

**Class**: `Luany\Lte\Parser`

Converts `.lte` source code into an AST (array of nodes).

**Token types**:
- `{{ $var }}` → `echo` node (HTML-escaped)
- `{!! $var !!}` → `raw_echo` node (unescaped)
- `@directive(args)` → `directive` node
- `{{-- comment --}}` → removed (not in AST)
- `@php ... @endphp` → `php_block` node
- Plain text → `text` node

### 7.3 Compiler

**Class**: `Luany\Lte\Compiler`

Compiles AST nodes into executable PHP strings.

**Custom directives**:

```php
$engine->getCompiler()->directive('datetime', function (?string $args) {
    return "<?php echo date({$args}); ?>";
});
```

### 7.4 SectionStack

**Class**: `Luany\Lte\SectionStack`

Static registry for layout sections and stacks.

| Method | Description |
|--------|-------------|
| `reset()` | Clear all state |
| `setLayout(string)` | Set parent layout |
| `getLayout(): ?string` | Get parent layout |
| `start(string)` | Begin capturing block section |
| `end()` | End block section capture |
| `set(string, string)` | Set inline section |
| `get(string, string): string` | Get section content |
| `has(string): bool` | Check if section exists |
| `startPush(string)` | Begin push to named stack |
| `endPush()` | End push capture |
| `getStack(string): string` | Render named stack |

### 7.5 AssetStack

**Class**: `Luany\Lte\AssetStack`

Manages inline `<style>` and `<script>` blocks with automatic deduplication.

| Method | Description |
|--------|-------------|
| `reset()` | Clear all state |
| `startStyle(array)` | Begin capturing style block |
| `endStyle()` | End style capture |
| `startScript(array)` | Begin capturing script block |
| `endScript()` | End script capture |
| `renderStyles(): string` | Render all `<style>` tags |
| `renderScripts(): string` | Render all `<script>` tags |
| `getStyleCount(): int` | Number of unique styles |
| `getScriptCount(): int` | Number of unique scripts |

---

## 8. luany/cli — Command-Line Tool

**Namespace**: `LuanyCli`

### 8.1 Architecture

```
LuanyCli\Application::run($argv)
    ↓
CommandRegistry → CommandInterface → handle($args)
```

All commands extend `BaseCommand` which implements `CommandInterface`.

### 8.2 CommandInterface

```php
interface CommandInterface
{
    public function name(): string;
    public function description(): string;
    public function handle(array $args): void;
    public function requiresProject(): bool;
}
```

### 8.3 Support Classes

| Class | Description |
|-------|-------------|
| `EnvParser` | Parse `.env` files manually (handles base64 `=` in values) |
| `FieldParser` | Convert field definitions to migration columns, form HTML, etc. |
| `ProjectFinder` | Walk directory tree to find nearest Luany project |
| `StubRenderer` | Replace `{{placeholders}}` in stub template files |

---

## 9. luany/luany — Application Skeleton

### 9.1 Bootstrap (`bootstrap/app.php`)

Sequence:
1. Load Composer autoloader
2. Create `Application` instance with base path
3. `Env::load()` + `Env::required(['APP_ENV', 'APP_URL'])`
4. Register `AppServiceProvider` and `DatabaseServiceProvider`
5. Bind custom `Kernel` and `Handler` as singletons
6. Return `$app`

### 9.2 Front Controller (`public/index.php`)

```php
$app      = require dirname(__DIR__) . '/bootstrap/app.php';
$kernel   = $app->make(Kernel::class);
$kernel->boot();
$request  = Request::fromGlobals();
$response = $kernel->handle($request);
$response->send();
$kernel->terminate($request, $response);
```

### 9.3 Default Providers

**AppServiceProvider** — Registers the LTE engine (`view`) and Translator (`translator`) in the container.

**DatabaseServiceProvider** — Configures `Connection::configure()` from `.env` DB credentials.

---

## 10. Configuration Reference

### `config/app.php`

| Key | Default | Description |
|-----|---------|-------------|
| `name` | `env('APP_NAME', 'Luany')` | Application name |
| `env` | `env('APP_ENV', 'production')` | Environment (development/production) |
| `debug` | `env('APP_DEBUG', false)` | Enable debug mode |
| `url` | `env('APP_URL', 'http://localhost:8000')` | Application URL |
| `locale` | `env('APP_LOCALE', 'en')` | Default locale |
| `fallback_locale` | `'en'` | Fallback when key missing |
| `supported_locales` | `['en', 'pt']` | Accepted locale codes |

### Adding a Language

1. Create `lang/{code}.php` returning a flat associative array
2. Add the code to `config/app.php` → `supported_locales`
3. Add a switch button in `views/components/navbar.lte`

---

## 11. Environment Variables

### Application

| Variable | Example | Required | Description |
|----------|---------|----------|-------------|
| `APP_NAME` | `"My App"` | No | Display name |
| `APP_ENV` | `development` | **Yes** | Environment |
| `APP_DEBUG` | `true` | No | Debug mode |
| `APP_URL` | `http://localhost:8000` | **Yes** | Base URL |
| `APP_KEY` | `base64:...` | No | Encryption key |
| `APP_TIMEZONE` | `UTC` | No | Timezone |
| `APP_LOCALE` | `en` | No | Default locale |
| `APP_HTTPS` | `false` | No | Force HTTPS |

### Database

| Variable | Example | Required | Description |
|----------|---------|----------|-------------|
| `DB_CONNECTION` | `mysql` | No | Driver |
| `DB_HOST` | `127.0.0.1` | No | Host |
| `DB_PORT` | `3306` | No | Port |
| `DB_NAME` | `luany` | No | Database name |
| `DB_USER` | `root` | No | Username |
| `DB_PASS` | *(empty)* | No | Password |

### Mail (optional)

| Variable | Example | Description |
|----------|---------|-------------|
| `MAIL_ENABLED` | `false` | Enable mail |
| `MAIL_HOST` | `smtp.gmail.com` | SMTP host |
| `MAIL_PORT` | `587` | SMTP port |
| `MAIL_ENCRYPTION` | `tls` | Encryption |
| `MAIL_USERNAME` | | SMTP user |
| `MAIL_PASSWORD` | | SMTP password |
| `MAIL_FROM_EMAIL` | | Sender email |
| `MAIL_FROM_NAME` | `${APP_NAME}` | Sender name |

---

## 12. CLI Command Reference

### Project

| Command | Description |
|---------|-------------|
| `luany new <name>` | Create a new Luany project |
| `luany serve [host] [port]` | Start PHP dev server (default: `localhost:8000`) |
| `luany key:generate` | Generate APP_KEY and write to `.env` |
| `luany cache:clear` | Clear compiled view cache |
| `luany about` | Display project and environment info |
| `luany doctor` | Full health check (PHP, extensions, DB, files) |
| `luany list` | List all available commands |

### Scaffolding

| Command | Description | Output |
|---------|-------------|--------|
| `luany make:controller <Name>` | Create controller | `app/Controllers/{Name}Controller.php` |
| `luany make:model <Name>` | Create model | `app/Models/{Name}.php` |
| `luany make:migration <name>` | Create migration | `database/migrations/{timestamp}_{name}.php` |
| `luany make:middleware <Name>` | Create middleware | `app/Http/Middleware/{Name}Middleware.php` |
| `luany make:provider <Name>` | Create provider | `app/Providers/{Name}ServiceProvider.php` |
| `luany make:view <name> [type]` | Create LTE view | `views/{path}.lte` (types: page, component, layout) |
| `luany make:feature <Name> [fields...]` | Full CRUD scaffold | Model + Controller + Migration + 4 Views + Routes |

**Feature scaffolding with inline fields**:

```bash
luany make:feature Product name:string price:decimal description:text active:boolean
```

Supported field types: `string`, `text`, `integer`, `boolean`, `email`, `date`, `decimal`.

### Migrations

| Command | Description |
|---------|-------------|
| `luany migrate` | Run all pending migrations |
| `luany migrate:rollback` | Rollback last batch |
| `luany migrate:fresh` | Drop all tables + re-run all migrations |
| `luany migrate:status` | Show migration status table |

### Subdirectory Support

Controllers and middleware support subdirectory paths:

```bash
luany make:controller Auth/Login
# → app/Controllers/Auth/LoginController.php
# → namespace App\Controllers\Auth

luany make:middleware Auth/Token
# → app/Http/Middleware/Auth/TokenMiddleware.php
```

---

## 13. LTE Template Reference

### Output

| Syntax | Compiled Output | Description |
|--------|-----------------|-------------|
| `{{ $var }}` | `htmlspecialchars($var)` | Escaped echo |
| `{!! $var !!}` | `echo $var` | Raw/unescaped echo |
| `{{-- comment --}}` | *(removed)* | Template comment |

### Conditionals

```html
@if($user)
    <p>Hello, {{ $user->name }}</p>
@elseif($guest)
    <p>Welcome, guest</p>
@else
    <p>Please login</p>
@endif

@unless($banned)
    <p>Welcome!</p>
@endunless

@isset($title)
    <h1>{{ $title }}</h1>
@endisset

@ifempty($items)
    <p>No items found.</p>
@endifempty
```

### Loops

```html
@foreach($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

@forelse($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>No users found.</p>
@endforelse

@for($i = 0; $i < 10; $i++)
    <span>{{ $i }}</span>
@endfor

@while($condition)
    ...
@endwhile
```

### Layout System

**Layout** (`views/layouts/main.lte`):

```html
<!DOCTYPE html>
<html lang="{{ locale() }}">
<head>
    <title>@yield('title', env('APP_NAME', 'Luany'))</title>
    @stack('head')
    @styles
</head>
<body>
    @yield('content')
    @scripts
    @stack('scripts')
</body>
</html>
```

**Child page** (`views/pages/home.lte`):

```html
@extends('layouts.main')

@section('title', 'Home Page')

@push('head')
    <meta name="description" content="Welcome">
@endpush

@section('content')
    <h1>Welcome</h1>
@endsection
```

### Includes

```html
@include('components.navbar')
@include('components.card', ['title' => 'Hello'])
```

Parent variables are automatically passed. Extra data can be merged via the second argument.

### Assets (Styles & Scripts)

```html
@style
    .card { padding: 1rem; }
@endstyle

@script(defer)
    document.querySelector('.card').addEventListener('click', fn);
@endscript
```

Place `@styles` in `<head>` and `@scripts` before `</body>` in your layout. Duplicate blocks (same content) are automatically deduplicated.

### Stacks

```html
{{-- In child views --}}
@push('head')
    <link rel="stylesheet" href="/css/page.css">
@endpush

{{-- In layout --}}
@stack('head')
```

Multiple `@push` calls to the same stack are accumulated (never replaced).

### PHP

```html
{{-- Inline --}}
@php($count = count($items))

{{-- Block --}}
@php
    $total = array_sum(array_column($items, 'price'));
@endphp
```

### Security

```html
<form method="POST">
    @csrf
    @method('PUT')
    ...
</form>
```

- `@csrf` → generates hidden input with `$_SESSION['csrf_token']`
- `@method('PUT')` → generates hidden `_method` input

### Auth Guards

```html
@auth
    <p>Welcome, logged in user!</p>
@endauth

@guest
    <a href="/login">Login</a>
@endguest
```

### Debug

```html
@dump($variable)    {{-- var_dump --}}
@dd($variable)      {{-- var_dump + die --}}
```

---

## 14. Integration Guide

### Creating a Service Provider

```php
namespace App\Providers;

use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;

class CacheServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton('cache', fn() => new FileCache(
            base_path('storage/cache')
        ));
    }
}
```

Register in `bootstrap/app.php`:

```php
$app->register(new App\Providers\CacheServiceProvider());
```

### Creating Custom Middleware

```php
namespace App\Http\Middleware;

use Luany\Core\Http\Request;
use Luany\Core\Http\Response;
use Luany\Core\Middleware\MiddlewareInterface;

class AuthMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response
    {
        if (!isset($_SESSION['user_id'])) {
            return Response::redirect('/login');
        }

        $response = $next($request);

        // After middleware logic here
        return $response;
    }
}
```

Add to `app/Http/Kernel.php`:

```php
protected array $middleware = [
    AuthMiddleware::class,
];
```

### Custom LTE Directives

```php
// In a service provider boot() method
$engine = app('view');
$engine->getCompiler()->directive('datetime', function (?string $args) {
    $format = $args ?: "'Y-m-d H:i'";
    return "<?php echo date({$format}); ?>";
});
```

Usage: `@datetime('d/m/Y')`

### Using the Query Builder Directly

```php
$builder = new \Luany\Database\QueryBuilder(
    \Luany\Database\Connection::getInstance()
);

$results = $builder->table('orders')
    ->select('id', 'total', 'created_at')
    ->where('status', '=', 'completed')
    ->orderBy('created_at', 'DESC')
    ->limit(25)
    ->get();
```

---

## 15. Upgrade Guide

### v0.2.x → v0.3.x (Framework)

**Breaking**: `Kernel::handle()` now uses a hybrid try/catch for proper middleware "after" phase execution on route exceptions.

**Action**: If you override `Kernel::handle()`, update your error handling logic.

### Security Fixes Applied (v0.1.3 Database, v0.2.4 Core, v0.3.1 Framework)

1. **SQL Injection in `Model::all()`** — `$orderBy` now validated against strict regex whitelist
2. **`$_GET` Pollution** — Router no longer writes route params into `$_GET`
3. **Uncaught Middleware Exceptions** — Kernel catches both route and middleware exceptions

**Action**: Update all three packages:

```bash
composer update luany/core luany/framework luany/database
```

---

*Generated for the Luany ecosystem at luany-ecosystem GitHub organization.*
*Last updated: 2026-03-19*
