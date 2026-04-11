# Luany Ecosystem — Comprehensive Documentation

> **Luany** — AST-compiled PHP MVC framework with an explicit request lifecycle and zero-regex template engine.

**Version**: 1.x &nbsp;|&nbsp; **PHP**: ≥ 8.2 &nbsp;|&nbsp; **License**: MIT
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
| **Core** | `luany/core` | HTTP Request/Response, Router, Middleware Pipeline, CORS, Rate Limiting | v1.0.0 |
| **Database** | `luany/database` | PDO Connection, Fluent QueryBuilder, Active Record ORM, Relations, Migrations, Seeders | v1.1.0 |
| **Framework** | `luany/framework` | Application container, Kernel, Service Providers, Config, Session, Validator, i18n | v1.0.0 |
| **LTE** | `luany/lte` | AST-compiled template engine (`.lte` files) | v1.0.0 |
| **CLI** | `luany/cli` | `luany` command-line tool, full CRUD scaffolding, migrations, seeders, LDE | v1.1.0 |
| **Skeleton** | `luany/luany` | Ready-to-use application template | v1.1.2 |

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
# With Luany CLI (recommended)
composer global require luany/cli
luany new my-app
cd my-app
luany doctor
luany migrate
luany dev
```

```bash
# Or directly via Composer
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

### Live Reload (Development)

```bash
npm install
luany dev
```

Starts the PHP built-in server and enables live reload via the **Luany Dev Engine (LDE)** — views, assets, and PHP files trigger instant browser updates with zero proxy layer.

**Reload strategy:**
- `.css` — CSS injected live, no full page reload
- `.lte` / `.php` / `.js` / `routes/` / `config/` — full page reload

> **Note:** `luany serve` continues to work as a plain PHP server without live reload.

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
       (Config, Session, CSRF, LTE engine, Routes auto-discovery)
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
│   │   └── Middleware/    # Application middleware (incl. DevMiddleware)
│   ├── Models/            # Active Record models
│   ├── Providers/         # Service providers
│   └── Support/           # Application helpers
├── bootstrap/
│   └── app.php            # Application bootstrap
├── config/
│   ├── app.php            # Application config
│   └── mail.php           # Mail config
├── database/
│   ├── migrations/        # Timestamped migrations
│   └── seeders/           # Database seeders
├── lang/                  # Translation files (en.php, pt.php)
├── public/
│   ├── index.php          # Front controller
│   └── assets/            # CSS, JS, images
├── routes/
│   ├── http.php           # Core route definitions
│   └── *.php              # Feature route files (auto-discovered)
├── storage/
│   ├── cache/views/       # Compiled LTE cache
│   └── logs/              # Log files
├── views/
│   ├── components/        # Reusable partials
│   ├── layouts/           # Layout templates
│   └── pages/             # Page views
├── .env                   # Environment config
├── package.json           # Node deps (chokidar + ws for LDE live reload)
└── composer.json
```

---

## 4. luany/core — HTTP & Routing

**Namespace**: `Luany\Core`

### 4.1 Request

**Class**: `Luany\Core\Http\Request`

Immutable HTTP request wrapper. Created via `Request::fromGlobals()` or manual construction.

```php
$request = Request::fromGlobals();

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
| `body` | `body(): array` | Body-only fields (excludes query string) |
| `all` | `all(): array` | Merged query + post params |
| `input` | `input(string $key, mixed $default = null): mixed` | Get a single input value |
| `query` | `query(string $key, mixed $default = null): mixed` | GET parameter |
| `post` | `post(string $key, mixed $default = null): mixed` | POST parameter |
| `cookie` | `cookie(string $name, ?string $default = null): ?string` | Cookie value |
| `server` | `server(string $key, mixed $default = null): mixed` | Server param |
| `only` | `only(array $keys): array` | Subset of inputs |
| `has` | `has(string $key): bool` | Check if key exists |
| `filled` | `filled(string $key): bool` | Key exists and is non-empty |
| `isMethod` | `isMethod(string $method): bool` | Check HTTP method |
| `isGet` / `isPost` | `isGet(): bool` / `isPost(): bool` | Method shortcuts |
| `isAjax` | `isAjax(): bool` | Check X-Requested-With header |
| `expectsJson` | `expectsJson(): bool` | Check Accept: application/json |
| `ip` | `ip(): string` | Client IP address |
| `routeParams` | `routeParams(): array` | Parameters extracted by router |
| `withRouteParams` | `withRouteParams(array $params): self` | Clone with route params |

### 4.2 Response

**Class**: `Luany\Core\Http\Response`

HTTP response builder with static factory methods.

```php
return Response::make('<h1>Hello</h1>', 200);
return Response::json(['user' => $user]);
return Response::redirect('/dashboard');
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
| `unauthorized` | `static unauthorized(): self` | 401 response |
| `forbidden` | `static forbidden(): self` | 403 response |
| `serverError` | `static serverError(): self` | 500 response |
| `header` | `header(string $name, string $value): self` | Add header (fluent) |
| `withHeaders` | `withHeaders(array $headers): self` | Add multiple headers |
| `withCookie` | `withCookie(string $name, string $value, array $options = []): self` | Set cookie |
| `status` | `status(int $code): self` | Set status code |
| `body` | `body(string $body): self` | Set body |
| `getStatusCode` | `getStatusCode(): int` | Get status code |
| `getBody` | `getBody(): string` | Get body content |
| `getHeaders` | `getHeaders(): array` | Get all headers |
| `isRedirect` | `isRedirect(): bool` | Is a redirect response |
| `isSuccessful` | `isSuccessful(): bool` | 2xx status |
| `send` | `send(): void` | Send to client |

### 4.3 Router

**Class**: `Luany\Core\Routing\Router`

```php
use Luany\Core\Routing\Route;

Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
Route::get('/users/{id}', [UserController::class, 'show']);
Route::put('/users/{id}', [UserController::class, 'update']);
Route::delete('/users/{id}', [UserController::class, 'destroy']);

// Resource routes — registers all 7 CRUD routes + PATCH alias
Route::resource('products', ProductController::class);

// Route groups with prefix and middleware
Route::group(['prefix' => '/admin', 'middleware' => [AuthMiddleware::class]], function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
});

// Named routes
Route::get('/users/{id}', [UserController::class, 'show'])->name('users.show');

// Route model binding
Route::bind('user', fn(string $id) => User::find($id));
Route::model('user', User::class); // uses User::find() automatically

// Route caching (production)
Route::cache(base_path('storage/cache/routes.php'));
Route::loadCache(base_path('storage/cache/routes.php'));
```

**Route Static Methods**:

| Method | Signature |
|--------|-----------|
| `get` | `static get(string $uri, array\|callable $action): RouteRegistrar` |
| `post` | `static post(string $uri, array\|callable $action): RouteRegistrar` |
| `put` | `static put(string $uri, array\|callable $action): RouteRegistrar` |
| `patch` | `static patch(string $uri, array\|callable $action): RouteRegistrar` |
| `delete` | `static delete(string $uri, array\|callable $action): RouteRegistrar` |
| `any` | `static any(string $uri, array\|callable $action): RouteRegistrar` |
| `resource` | `static resource(string $name, string $controller): void` |
| `apiResource` | `static apiResource(string $name, string $controller): void` |
| `group` | `static group(array $attributes, callable $callback): void` |
| `bind` | `static bind(string $param, callable $resolver): void` |
| `model` | `static model(string $param, string $modelClass): void` |
| `view` | `static view(string $uri, string $view, array $data = []): void` |
| `cache` | `static cache(string $path): void` | Save compiled routes |
| `loadCache` | `static loadCache(string $path): bool` | Load cached routes |
| `clearCache` | `static clearCache(string $path): void` | Delete route cache |
| `reset` | `static reset(): void` | Reset router state (testing) |
| `getRoutes` | `static getRoutes(): array` | Get all registered routes |

### 4.4 MiddlewareInterface

```php
interface MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response;
}
```

### 4.5 Pipeline

```php
$response = (new Pipeline())
    ->send($request)
    ->through([AuthMiddleware::class, LogMiddleware::class])
    ->then(fn(Request $req) => $router->handle($req));
```

### 4.6 CorsMiddleware

**Class**: `Luany\Core\Middleware\CorsMiddleware`

Configurable CORS middleware.

```php
$cors = new CorsMiddleware([
    'allowed_origins'  => ['https://app.example.com', '*.example.com'],
    'allowed_methods'  => ['GET', 'POST', 'PUT', 'DELETE'],
    'allowed_headers'  => ['Content-Type', 'Authorization'],
    'exposed_headers'  => [],
    'max_age'          => 86400,
    'allow_credentials'=> false,
]);
```

### 4.7 Rate Limiting

**Interface**: `Luany\Core\RateLimit\RateLimiterInterface`

```php
// In-memory (testing/single process)
$limiter = new InMemoryRateLimiter();

// File-based (multi-process)
$limiter = new FileRateLimiter(storage_path('rate-limits'));

$limiter->attempt('key', maxAttempts: 60, decaySeconds: 60);
$limiter->remaining('key', maxAttempts: 60);
$limiter->tooManyAttempts('key', maxAttempts: 60);
$limiter->reset('key');
```

**RateLimitMiddleware**:

```php
// In routes/http.php
Route::group(['middleware' => [new RateLimitMiddleware($limiter, 60, 60)]], function () {
    Route::post('/api/login', [AuthController::class, 'login']);
});
```

### 4.8 Exceptions

| Class | Code | Description |
|-------|------|-------------|
| `RouteNotFoundException` | 404 | No route matched the request |
| `MethodNotAllowedException` | 405 | URI matched but HTTP method did not — includes `Allow` header |

---

## 5. luany/database — ORM & Migrations

**Namespace**: `Luany\Database`

### 5.1 Connection

**Class**: `Luany\Database\Connection`

```php
// Configure (called by DatabaseServiceProvider)
Connection::configure([
    'host'    => '127.0.0.1',
    'port'    => 3306,
    'dbname'  => 'my_app',
    'user'    => 'root',
    'pass'    => '',
    'charset' => 'utf8mb4',
]);

// Transaction support
$connection->beginTransaction();
$connection->commit();
$connection->rollBack();

// Transaction callback (auto commit/rollback)
$connection->transaction(function (\PDO $pdo) {
    // operations
});
```

### 5.2 QueryBuilder

**Class**: `Luany\Database\QueryBuilder`

Fully fluent query builder with prepared statements.

```php
$qb = new QueryBuilder($connection);

// SELECT with full fluent chain
$users = $qb->table('users')
    ->select('id', 'name', 'email')
    ->where('active', '=', 1)
    ->orWhere('role', '=', 'admin')
    ->whereIn('status', ['active', 'pending'])
    ->whereNotNull('email_verified_at')
    ->orderBy('name', 'ASC')
    ->orderBy('created_at', 'DESC')
    ->limit(10)
    ->offset(20)
    ->get();

// Existence check
$exists = $qb->table('users')->where('email', '=', $email)->exists();

// Pagination
$result = $qb->table('products')->paginate(perPage: 15, page: 2);
// Returns PaginationResult

// Raw query (backward-compatible)
$result = $qb->query('SELECT * FROM users WHERE active = ?', [1]);
$result = $qb->raw('SELECT COUNT(*) FROM orders WHERE total > ?', [100]);
```

**Fluent Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `table` | `table(string $table): static` | Set target table (returns fresh builder) |
| `select` | `select(string ...$columns): static` | Columns to select |
| `where` | `where(string $col, string $op, mixed $val): static` | AND WHERE clause |
| `orWhere` | `orWhere(string $col, string $op, mixed $val): static` | OR WHERE clause |
| `whereIn` | `whereIn(string $col, array $vals): static` | WHERE IN |
| `whereNull` | `whereNull(string $col): static` | WHERE IS NULL |
| `whereNotNull` | `whereNotNull(string $col): static` | WHERE IS NOT NULL |
| `orderBy` | `orderBy(string $col, string $dir = 'ASC'): static` | ORDER BY |
| `limit` | `limit(int $limit): static` | LIMIT |
| `offset` | `offset(int $offset): static` | OFFSET |
| `get` | `get(): array` | Execute SELECT, return rows |
| `first` | `first(): ?array` | First row or null |
| `insert` | `insert(array $data): bool` | Insert row |
| `update` | `update(array $data): int` | Update rows, return count |
| `delete` | `delete(): int` | Delete rows, return count |
| `count` | `count(): int` | Count matching rows |
| `exists` | `exists(): bool` | Check if any row matches |
| `paginate` | `paginate(int $perPage = 15, int $page = 1): PaginationResult` | Paginated results |
| `raw` | `raw(string $sql, array $bindings = []): Result` | Raw SELECT |
| `query` | `query(string $sql, array $bindings = []): Result` | Alias for raw |
| `statement` | `statement(string $sql, array $bindings = []): int` | Raw INSERT/UPDATE/DELETE |
| `lastInsertId` | `lastInsertId(): string` | Last inserted ID |

### 5.3 PaginationResult

**Class**: `Luany\Database\PaginationResult`

```php
$pagination = User::query()->paginate(15, page: 2);

$pagination->data;        // array of rows
$pagination->total;       // total record count
$pagination->perPage;     // records per page
$pagination->currentPage; // current page number
$pagination->lastPage;    // last page number
$pagination->from;        // first record number on this page
$pagination->to;          // last record number on this page
$pagination->hasMore();   // bool — more pages exist
$pagination->hasPrev();   // bool — previous page exists
$pagination->toArray();   // full array representation
```

### 5.4 Result

**Class**: `Luany\Database\Result`

| Method | Signature | Description |
|--------|-----------|-------------|
| `fetchAll` | `fetchAll(): array` | All rows as associative arrays |
| `fetchOne` | `fetchOne(): ?array` | First row or null |
| `fetchColumn` | `fetchColumn(int $index = 0): array` | Single column values |
| `rowCount` | `rowCount(): int` | Number of affected/returned rows |

### 5.5 Model

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
    protected array  $casts      = ['active' => 'bool', 'score' => 'int'];
}
```

**Usage**:

```php
// Find
$user = User::find(42);

// All records
$users = User::all('created_at DESC');

// Filtered
$admins = User::where('role = ?', ['admin']);
$first  = User::firstWhere('email = ?', [$email]);

// Count
$total = User::count();
$count = User::count('role = ?', ['admin']);

// Paginate
$page = User::newQuery()->where('active', '=', 1)->paginate(15, 1);

// Create
$user = User::create(['name' => 'Maria', 'email' => 'maria@example.com']);

// Save (INSERT if new, UPDATE if existing)
$user = new User();
$user->name = 'João';
$user->save();

// Update
$user->update(['name' => 'João Silva']);

// Delete
$user->delete();

// Serialize
$array = $user->toArray(); // respects $hidden
$json  = $user->toJson();
```

**Model Methods**:

| Method | Signature | Description |
|--------|-----------|-------------|
| `find` | `static find(mixed $id): ?static` | Find by primary key |
| `all` | `static all(string $orderBy = ''): array<int, static>` | All records |
| `where` | `static where(string $conditions, array $bindings = []): array<int, static>` | Raw WHERE |
| `firstWhere` | `static firstWhere(string $conditions, array $bindings = []): ?static` | First match |
| `count` | `static count(string $conditions = '', array $bindings = []): int` | Count |
| `newQuery` | `static newQuery(): QueryBuilder` | Fresh QueryBuilder scoped to table |
| `create` | `static create(array $attributes): static` | Insert and return |
| `save` | `save(): bool` | Persist |
| `update` | `update(array $attributes): bool` | Update this record |
| `delete` | `delete(): bool` | Delete this record |
| `fill` | `fill(array $attributes): void` | Mass-assign fillable |
| `toArray` | `toArray(): array` | Array (excludes hidden) |
| `toJson` | `toJson(): string` | JSON (excludes hidden) |
| `exists` | `exists(): bool` | Was loaded from DB |
| `getTable` | `getTable(): string` | Table name |
| `getPrimaryKey` | `getPrimaryKey(): string` | Primary key name |

### 5.6 Relationships

```php
class User extends Model
{
    public function profile(): ?Profile
    {
        return $this->hasOne(Profile::class, 'user_id');
    }

    public function posts(): array
    {
        return $this->hasMany(Post::class, 'user_id');
    }
}

class Post extends Model
{
    public function user(): ?User
    {
        return $this->belongsTo(User::class, 'user_id');
    }
}
```

**Eager loading** — prevents N+1 queries:

```php
// Load users with posts pre-fetched in a single WHERE IN query
$users = User::with('posts')->all();

foreach ($users as $user) {
    foreach ($user->posts as $post) { // no extra query
        echo $post->title;
    }
}
```

**Relation methods**:

| Method | Description |
|--------|-------------|
| `hasOne(related, foreignKey, localKey)` | 1:1 — related table has FK pointing here |
| `hasMany(related, foreignKey, localKey)` | 1:N — related table has FK pointing here |
| `belongsTo(related, foreignKey, ownerKey)` | Inverse — this table has FK pointing to related |
| `with(string ...$relations): static` | Static — enable eager loading for next query |
| `getRelation(string $relation): mixed` | Get cached or lazy-loaded relation |
| `setRelation(string $relation, mixed $value): void` | Set relation value |

### 5.7 SoftDeletes

**Trait**: `Luany\Database\Concerns\SoftDeletes`

```php
class Post extends Model
{
    use SoftDeletes;
    // Requires `deleted_at TIMESTAMP NULL` column in migration
}

$post->delete();         // sets deleted_at, does NOT physically delete
$post->trashed();        // true if soft-deleted
$post->restore();        // clears deleted_at
$post->forceDelete();    // physically deletes

Post::withTrashed();     // includes soft-deleted in queries
Post::onlyTrashed();     // only soft-deleted records
```

### 5.8 Migration System

```php
use Luany\Database\Migration\Migration;

class CreateUsersTable extends Migration
{
    public function up(\PDO $pdo): void
    {
        $pdo->exec("
            CREATE TABLE IF NOT EXISTS `users` (
                `id`         INT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
                `name`       VARCHAR(255) NOT NULL,
                `email`      VARCHAR(150) NOT NULL UNIQUE,
                `deleted_at` TIMESTAMP NULL DEFAULT NULL,
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

### 5.9 Seeder System

Seeders live in `database/seeders/` and populate the database with initial or test data. Run with `luany db:seed`.

```php
use Luany\Database\Seeder\Seeder;

// Entry point — chains all seeders
class DatabaseSeeder extends Seeder
{
    public function run(\PDO $pdo): void
    {
        $this->call(UserSeeder::class);
    }
}

// Individual seeder
class UserSeeder extends Seeder
{
    public function run(\PDO $pdo): void
    {
        $stmt = $pdo->prepare("INSERT IGNORE INTO `users` (`name`, `email`) VALUES (?, ?)");
        $stmt->execute(['António Ngola', 'antonio@example.com']);
    }
}
```

**SeederRunner** — used internally by the CLI:

```php
use Luany\Database\Seeder\SeederRunner;

$runner = new SeederRunner($pdo, '/path/to/database/seeders');
$runner->run('DatabaseSeeder');           // default entry point
$runner->run('UserSeeder');               // specific seeder
```

---

## 6. luany/framework — Application Layer

**Namespace**: `Luany\Framework`

### 6.1 Application

**Class**: `Luany\Framework\Application`

IoC container. Implements `ApplicationInterface`.

```php
$app = new Application('/path/to/project');

$app->singleton('db', fn($app) => new Connection($config));
$app->bind('logger', fn($app) => new Logger());

$db = $app->make('db');
$app->register(new AppServiceProvider());
```

### 6.2 HTTP Kernel

**Class**: `Luany\Framework\Http\Kernel`

```php
class Kernel extends \Luany\Framework\Http\Kernel
{
    protected array $middleware = [
        DevMiddleware::class,    // FIRST — dev-only LDE live reload (zero-cost in production)
        LocaleMiddleware::class,
        CsrfMiddleware::class,
    ];
    // No $routesFile override needed — auto-discovery loads all routes/*.php
}
```

**Route auto-discovery**: Kernel loads `routes/http.php` first, then all other `*.php` files in the `routes/` directory alphabetically. `make:feature` generates per-feature route files that are picked up automatically.

### 6.3 Config

**Class**: `Luany\Framework\Support\Config`

Dot-notation config loader from PHP files in `config/`.

```php
config('app.name');           // 'My App'
config('app.debug');          // true/false
config('app.locale', 'en');   // with default
config()->set('app.name', 'New Name');
config()->has('app.debug');   // bool
config()->all();              // full array
```

### 6.4 Session

**Interface**: `Luany\Framework\Contracts\SessionInterface`
**Class**: `Luany\Framework\Session\FileSession`

File-based session driver with flash data aging.

```php
session()->set('user_id', 42);
session()->get('user_id');
session()->has('user_id');
session()->forget('user_id');
session()->flash('status', 'Profile updated.');
session()->regenerate();
session()->all();
session()->destroy();
```

**Flash data lifecycle**:
- `flash('key', $value)` — stored as "new"
- Next request: "new" becomes "old" — available via `get('key')`
- Request after: "old" is purged

### 6.5 CSRF Protection

**Class**: `Luany\Framework\Security\CsrfToken`

```php
csrf_token();    // Get or generate token
// In forms: @csrf generates <input type="hidden" name="csrf_token" value="...">
// In AJAX: X-CSRF-Token header
```

**CsrfMiddleware** — protects POST/PUT/PATCH/DELETE. Exempt routes via `$except` list.

### 6.6 Validator

**Class**: `Luany\Framework\Validation\Validator`

Zero-dependency validation engine.

```php
$validator = Validator::make($request->body(), [
    'name'     => 'required|string|min:2|max:255',
    'email'    => 'required|email|unique:users,email',
    'password' => 'required|string|min:8|confirmed',
    'role'     => 'required|in:admin,editor,viewer',
    'age'      => 'required|numeric|min:18',
    'active'   => 'boolean',
]);

if ($validator->fails()) {
    $errors = $validator->errors(); // array<string, array<string>>
}

$validated = $validator->validated(); // only validated fields
```

Rules also accept array format:

```php
'name' => ['required', 'string', 'max:255'],
```

**Supported rules**:

| Rule | Description |
|------|-------------|
| `required` | Field must be present and non-empty |
| `string` | Must be a string |
| `email` | Must be a valid email |
| `numeric` | Must be numeric |
| `integer` | Must be an integer |
| `min:{n}` | Min length (string) or value (numeric) |
| `max:{n}` | Max length (string) or value (numeric) |
| `in:{a},{b}` | Must be one of the listed values |
| `confirmed` | Must have matching `{field}_confirmation` |
| `boolean` | Must be boolean-like |
| `unique:{table},{col}` | Must be unique (callback-based) |

### 6.7 Exception Handler

**Class**: `Luany\Framework\Exceptions\Handler` (abstract)

```php
class Handler extends \Luany\Framework\Exceptions\Handler
{
    public function render(\Throwable $e): Response
    {
        if ($e instanceof ModelNotFoundException) {
            return Response::notFound();
        }
        return parent::render($e);
    }
}
```

### 6.8 HttpException & ValidationException

```php
// abort() throws HttpException — caught by Kernel
abort(404);
abort(403, 'Access denied.');
abort(422, 'Unprocessable content.');

// validate() throws ValidationException on failure — Kernel redirects back with errors
$data = validate($request->body(), [
    'name'  => 'required|string',
    'email' => 'required|email',
], '/back-url');
```

### 6.9 Env

**Class**: `Luany\Framework\Support\Env`

```php
Env::load('/path/to/project');
$host  = Env::get('DB_HOST', 'localhost');
$debug = Env::get('APP_DEBUG', false); // cast to bool
Env::required(['DB_HOST', 'DB_NAME']);
```

**Type casting**: `'true'` → `true`, `'false'` → `false`, `'null'` → `null`, `'empty'` → `''`.

### 6.10 Translator

**Class**: `Luany\Framework\Support\Translator`

```php
__('nav.home');
__('footer.copyright', ['year' => '2026']);
```

**Translation file** (`lang/en.php`):

```php
return [
    'nav.home'         => 'Home',
    'footer.copyright' => '© :year Luany',
];
```

### 6.11 Global Helper Functions

| Function | Signature | Description |
|----------|-----------|-------------|
| `app()` | `app(?string $abstract = null): mixed` | Application instance or resolve |
| `env()` | `env(string $key, mixed $default = null): mixed` | Environment variable |
| `config()` | `config(?string $key = null, mixed $default = null): mixed` | Config value |
| `session()` | `session(): SessionInterface` | Session service |
| `csrf_token()` | `csrf_token(): string` | Current CSRF token |
| `old()` | `old(string $key, mixed $default = null): mixed` | Previous input (after validation failure) |
| `base_path()` | `base_path(string $path = ''): string` | Absolute path to project root |
| `view()` | `view(string $name, array $data = []): string` | Render LTE view |
| `redirect()` | `redirect(string $url, int $status = 302): Response` | Redirect response |
| `response()` | `response(string $body = '', int $status = 200): Response` | Create response |
| `abort()` | `abort(int $code, string $message = ''): never` | Throw HttpException |
| `validate()` | `validate(array $data, array $rules, string $back): array` | Validate or redirect |
| `__()` | `__(string $key, array $replace = []): string` | Translate key |
| `locale()` | `locale(): string` | Current locale code |

---

## 7. luany/lte — Template Engine

**Namespace**: `Luany\Lte`

LTE (Luany Template Engine) compiles `.lte` templates into cached PHP files via AST parsing.

### 7.1 Engine

```php
$engine = new Engine(
    viewsPath: '/path/to/views',
    cachePath: '/path/to/cache',
    autoReload: true,
);

$html = $engine->render('pages.home', ['title' => 'Welcome']);

// Custom directives
$engine->getCompiler()->directive('datetime', function (?string $args) {
    return "<?php echo date({$args}); ?>";
});
```

### 7.2 SectionStack

| Method | Description |
|--------|-------------|
| `reset()` | Clear all state |
| `setLayout(string)` | Set parent layout |
| `start(string)` | Begin capturing section |
| `end()` | End section capture |
| `get(string, string): string` | Get section content |
| `startPush(string)` | Begin push to named stack |
| `endPush()` | End push capture |
| `getStack(string): string` | Render named stack |

### 7.3 AssetStack

Manages inline `<style>` and `<script>` blocks with automatic deduplication.

| Method | Description |
|--------|-------------|
| `startStyle(array)` | Begin capturing style block |
| `endStyle()` | End style capture |
| `startScript(array)` | Begin capturing script block |
| `endScript()` | End script capture |
| `renderStyles(): string` | Render all `<style>` tags |
| `renderScripts(): string` | Render all `<script>` tags |

---

## 8. luany/cli — Command-Line Tool

**Namespace**: `LuanyCli`

### 8.1 Architecture

```
LuanyCli\Application::run($argv)
    ↓
CommandRegistry → CommandInterface → handle($args)
```

### 8.2 Support Classes

| Class | Description |
|-------|-------------|
| `EnvParser` | Parse `.env` files (handles base64 `=` in values) |
| `FieldParser` | Convert field definitions to migration columns, form HTML, casts, validation rules |
| `ProjectFinder` | Walk directory tree to find nearest Luany project |
| `StubRenderer` | Replace `{{placeholders}}` in stub files |

### 8.3 Dev Classes (LDE)

| Class | Description |
|-------|-------------|
| `Dev\ProcessManager` | Orchestrates PHP server + Node watcher via `proc_open()`. Tick loop, clean shutdown (SIGTERM → SIGKILL) |
| `Dev\NodeRunner` | Spawns the Node.js watcher process. Verifies `node`, `chokidar`, `ws`, `watcher.js` |

**Resources**:

| File | Description |
|------|-------------|
| `Resources/dev/watcher.js` | Chokidar watcher + WebSocket server. Debounce 40ms. CSS → inject, PHP/LTE/JS → reload |
| `Resources/dev/client.js` | Browser WebSocket client. CSS inject with cache-buster. Exponential back-off reconnect |

---

## 9. luany/luany — Application Skeleton

### 9.1 Bootstrap (`bootstrap/app.php`)

```php
// Sequence:
// 1. Load Composer autoloader
// 2. new Application(base path)
// 3. Env::load() + Env::required(['APP_ENV', 'APP_URL'])
// 4. Register AppServiceProvider, DatabaseServiceProvider
// 5. Bind Kernel and Handler as singletons
// 6. Return $app
```

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

### 9.3 Application Helpers (`app/Support/helpers.php`)

| Function | Description |
|----------|-------------|
| `url(string $path)` | Absolute URL |
| `asset(string $path)` | Asset URL |
| `route(string $name, array $params)` | Named route URL |
| `flash(string $type, string $message)` | Set flash message |
| `get_flash(): ?array` | Get flash message |
| `auth_user(): ?int` | Current user ID |
| `is_authenticated(): bool` | Is user logged in |
| `e(mixed $value): string` | HTML escape |
| `csrf_field(): string` | Hidden CSRF input HTML |

---

## 10. Configuration Reference

### `config/app.php`

| Key | Default | Description |
|-----|---------|-------------|
| `name` | `env('APP_NAME', 'Luany')` | Application name |
| `env` | `env('APP_ENV', 'production')` | Environment |
| `debug` | `env('APP_DEBUG', false)` | Debug mode |
| `url` | `env('APP_URL', 'http://localhost:8000')` | Application URL |
| `locale` | `env('APP_LOCALE', 'en')` | Default locale |
| `fallback_locale` | `'en'` | Fallback locale |
| `supported_locales` | `['en', 'pt']` | Supported locales |

---

## 11. Environment Variables

### Application

| Variable | Example | Required | Description |
|----------|---------|----------|-------------|
| `APP_NAME` | `"My App"` | No | Display name |
| `APP_ENV` | `development` | **Yes** | Environment (`development` activates DevMiddleware) |
| `APP_DEBUG` | `true` | No | Debug mode |
| `APP_URL` | `http://localhost:8000` | **Yes** | Base URL |
| `APP_KEY` | `base64:...` | No | Encryption key |
| `APP_TIMEZONE` | `UTC` | No | Timezone |
| `APP_LOCALE` | `en` | No | Default locale |

### Database

| Variable | Example | Description |
|----------|---------|-------------|
| `DB_HOST` | `127.0.0.1` | MySQL host |
| `DB_PORT` | `3306` | MySQL port |
| `DB_NAME` | `luany` | Database name |
| `DB_USER` | `root` | Username |
| `DB_PASS` | *(empty)* | Password |

### Mail (optional)

| Variable | Example |
|----------|---------|
| `MAIL_ENABLED` | `false` |
| `MAIL_HOST` | `smtp.gmail.com` |
| `MAIL_PORT` | `587` |
| `MAIL_ENCRYPTION` | `tls` |
| `MAIL_USERNAME` | |
| `MAIL_PASSWORD` | |
| `MAIL_FROM_EMAIL` | |
| `MAIL_FROM_NAME` | `${APP_NAME}` |

---

## 12. CLI Command Reference

### Global — run anywhere

| Command | Description |
|---------|-------------|
| `luany new <name>` | Create a new Luany project |
| `luany doctor` | Full environment & project health check |
| `luany about` | Display project and environment info |
| `luany list` | List all available commands |

### Project — require a valid Luany project

| Command | Description |
|---------|-------------|
| `luany serve` | Start PHP dev server (`localhost:8000`) — no live reload |
| `luany dev` | Start Luany Dev Engine — PHP server + live reload (LDE) |
| `luany key:generate` | Generate APP_KEY and write to `.env` |
| `luany cache:clear` | Clear compiled view cache |
| `luany route:list` | Display all registered routes in a table |

### luany dev

```bash
luany dev
luany dev localhost 8080          # custom host/port
luany dev localhost 8000 35730    # custom WebSocket port
```

**Requirements:** Node.js on PATH · `npm install` run · `APP_ENV=development` in `.env`

**Architecture:**
```
Browser ←──────────────────→ PHP   (port 8000) — direct, no proxy
Browser ←── WebSocket ──────→ Node (port 35729) — reload signals only
```

DevMiddleware intercepts HTML responses and injects the LDE browser client before `</body>`. The client connects to the WebSocket server and applies live changes without a full page reload for CSS.

### Scaffolding

| Command | Output |
|---------|--------|
| `luany make:controller <Name>` | `app/Controllers/{Name}Controller.php` |
| `luany make:model <Name>` | `app/Models/{Name}.php` |
| `luany make:migration <name>` | `database/migrations/{timestamp}_{name}.php` |
| `luany make:middleware <Name>` | `app/Http/Middleware/{Name}Middleware.php` |
| `luany make:provider <Name>` | `app/Providers/{Name}ServiceProvider.php` |
| `luany make:view <name> [type]` | `views/{path}.lte` (types: `page`, `component`, `layout`) |
| `luany make:request <Name>` | `app/Http/Requests/{Name}Request.php` |
| `luany make:test <Name>` | `tests/{Name}Test.php` |
| `luany make:feature <Name> [fields...]` | Model + Controller + Migration + 4 Views + Routes |
| `luany make:seeder <Name>` | `database/seeders/{Name}Seeder.php` |

**`luany make:feature`** — interactive or inline:

```bash
# Interactive
luany make:feature Product

# Inline (no prompts)
luany make:feature Product name:string price:decimal active:boolean
```

Supported field types: `string`, `text`, `integer`, `boolean`, `email`, `date`, `decimal`.

Generated files per feature:

```
app/Models/Product.php                          ← $fillable, $casts
app/Controllers/ProductController.php           ← full CRUD (index, show, create, store, edit, update, destroy)
database/migrations/{ts}_create_products_table.php
views/pages/products/index.lte                  ← table with View/Edit/Delete actions
views/pages/products/show.lte                   ← detail card
views/pages/products/create.lte                 ← form with validation
views/pages/products/edit.lte                   ← form pre-filled
routes/products.php                             ← Route::resource('products', ...)
```

**Field type mapping**:

| Type | Migration | Form input | Cast | Behaviour |
|------|-----------|------------|------|-----------|
| `string` | `VARCHAR(255)` | `text` | — | `required`, `placeholder` |
| `text` | `TEXT` | `textarea` | — | `required`, `placeholder` |
| `integer` | `INT` | `number` | `int` | `required`, `placeholder` |
| `boolean` | `TINYINT(1)` | `checkbox` | `bool` | — |
| `email` | `VARCHAR(150)` | `email` | — | `required`, `placeholder` |
| `date` | `DATE` | `date` | — | `required`, `placeholder` |
| `decimal` | `DECIMAL(10,2)` | `number step="0.01"` | `float` | `required`, `placeholder` |

### Subdirectory support

```bash
luany make:controller Auth/Login     # → app/Controllers/Auth/LoginController.php
luany make:middleware Auth/Token     # → app/Http/Middleware/Auth/TokenMiddleware.php
```

### Migrations

| Command | Description |
|---------|-------------|
| `luany migrate` | Run all pending migrations |
| `luany migrate:rollback` | Rollback last batch |
| `luany migrate:fresh` | Drop all tables + re-run all |
| `luany migrate:fresh --seed` | Drop all tables + re-run all + seed |
| `luany migrate:status` | Show migration status table |

### Seeders

| Command | Description |
|---------|-------------|
| `luany make:seeder <Name>` | Scaffold a new seeder class |
| `luany db:seed` | Run `DatabaseSeeder` (default entry point) |
| `luany db:seed --class=Name` | Run a specific seeder |
| `luany migrate:fresh --seed` | Drop all tables, re-migrate, then seed |

---

## 13. LTE Template Reference

### Output

| Syntax | Output | Description |
|--------|--------|-------------|
| `{{ $var }}` | `htmlspecialchars($var)` | Escaped echo |
| `{!! $var !!}` | `echo $var` | Raw/unescaped echo |
| `{{-- comment --}}` | *(removed)* | Template comment |

### Conditionals

```lte
@if($user)
    <p>Hello, {{ $user->name }}</p>
@elseif($guest)
    <p>Welcome, guest</p>
@else
    <p>Please login</p>
@endif

@unless($banned)   <p>Welcome!</p>   @endunless
@isset($title)     <h1>{{ $title }}</h1>   @endisset
@ifempty($items)   <p>No items.</p>   @endifempty
```

### Loops

```lte
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
```

### Layout System

**Layout** (`views/layouts/main.lte`):

```lte
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

**Child page**:

```lte
@extends('layouts.main')

@section('title')My Page@endsection

@push('head')
    <meta name="description" content="...">
@endpush

@section('content')
    <h1>Welcome</h1>
@endsection
```

### Includes

```lte
@include('components.navbar')
@include('components.card', ['title' => 'Hello'])
```

### Components & Slots

```lte
{{-- Define a component: views/components/card.lte --}}
<div class="card">
    @slot('header')
        Default header
    @endslot

    <div class="body">
        @yield('default')
    </div>
</div>

{{-- Use the component --}}
@component('components.card')
    @slot('header')
        Custom Header
    @endslot
    <p>Card body content</p>
@endcomponent
```

### Assets (Styles & Scripts)

```lte
@style
    .card { padding: 1rem; background: var(--color-bg-card); }
@endstyle

@script(defer)
    document.querySelector('.card').addEventListener('click', fn);
@endscript
```

Place `@styles` in `<head>` and `@scripts` before `</body>`. Duplicate blocks are automatically deduplicated.

### Stacks

```lte
{{-- Child view --}}
@push('head')
    <link rel="stylesheet" href="/css/page.css">
@endpush

{{-- Layout --}}
@stack('head')
```

### PHP

```lte
@php($count = count($items))

@php
    $total = array_sum(array_column($items, 'price'));
@endphp
```

### Security

```lte
<form method="POST">
    @csrf
    @method('PUT')
    ...
</form>
```

### New in v1.0

```lte
{{-- @json — safe JSON output --}}
<script>
    const data = @json($items);
</script>

{{-- @class — conditional CSS classes --}}
<div @class(['active' => $isActive, 'disabled' => $isDisabled, 'card' => true])>
    ...
</div>
```

### Auth Guards

```lte
@auth
    <p>Welcome, logged in user!</p>
@endauth

@guest
    <a href="/login">Login</a>
@endguest
```

### Debug

```lte
@dump($variable)
@dd($variable)
```

---

## 14. Integration Guide

### Service Provider

```php
class CacheServiceProvider extends ServiceProvider
{
    public function register(Application $app): void
    {
        $app->singleton('cache', fn() => new FileCache(base_path('storage/cache')));
    }
}
```

Register in `bootstrap/app.php`:

```php
$app->register(new App\Providers\CacheServiceProvider());
```

### Custom Middleware

```php
class AuthMiddleware implements MiddlewareInterface
{
    public function handle(Request $request, callable $next): Response
    {
        if (!is_authenticated()) {
            return Response::redirect('/login');
        }
        return $next($request);
    }
}
```

### Custom LTE Directives

```php
// In a service provider boot()
app('view')->getCompiler()->directive('datetime', function (?string $args) {
    $format = $args ?: "'Y-m-d H:i'";
    return "<?php echo date({$format}); ?>";
});
```

Usage: `@datetime('d/m/Y')`

### Form Validation Pattern

```php
public function store(Request $request): Response
{
    $data = validate($request->body(), [
        'name'  => 'required|string|max:255',
        'email' => 'required|email',
        'price' => 'required|numeric',
    ], '/products/create');

    Product::create($data);
    flash('success', 'Product created successfully.');
    return redirect('/products');
}
```

In the view — display validation errors and old input:

```lte
@if(session()->get('errors'))
    @foreach(session()->get('errors') as $field => $messages)
        <p class="error">{{ $messages[0] }}</p>
    @endforeach
@endif

<input type="text" name="name" value="{{ old('name') }}">
```

---

## 15. Upgrade Guide

### v0.x → v1.0

See `UPGRADE.md` in the skeleton repository for the full migration guide.

**Quick summary of breaking changes**:

| Area | Change | Action |
|------|--------|--------|
| `abort()` | Moved to `luany/framework` — throws `HttpException` | Remove custom helper |
| `csrf_token()` | Session key corrected (`csrf_token`) | Remove custom override |
| Session bootstrap | `Kernel` manages session — no manual `session_start()` | Remove from `AppServiceProvider` |
| Route files | Auto-discovery enabled (`routes/*.php`) | No action required |
| `make:feature` | Generates `routes/{slug}.php` instead of appending to `http.php` | No action required |
| PHP requirement | `>=8.1` raised to `>=8.2` | Update server/CI |

### v1.0 → v1.1 (skeleton only)

| Area | Change | Action |
|------|--------|--------|
| `npm run dev` | Replaced by `luany dev` (LDE) | Run `npm install`, use `luany dev` |
| `DevMiddleware` | Added to skeleton — register as first middleware | Add `DevMiddleware::class` first in `Kernel::$middleware` |
| `package.json` | BrowserSync replaced by `chokidar` + `ws` | Run `npm install` after updating |

**Update packages**:

```bash
composer update luany/core luany/framework luany/database luany/lte
composer update luany/cli
```

---

*Last updated: 2026-04-11 — luany/cli v1.1.0 · luany/luany v1.1.2 · luany/database v1.1.0*