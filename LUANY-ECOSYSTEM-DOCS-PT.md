# Ecossistema Luany — Documentação Completa

> **Luany** — Framework PHP MVC compilado por AST com ciclo de vida de requisição explícito e motor de templates sem regex.

**Versão**: 1.x &nbsp;|&nbsp; **PHP**: ≥ 8.2 &nbsp;|&nbsp; **Licença**: MIT
**Autor**: António Ambrósio Ngola &nbsp;|&nbsp; **Org**: [luany-ecosystem](https://github.com/luany-ecosystem)

---

## Índice

1. [Visão Geral dos Pacotes](#1-visão-geral-dos-pacotes)
2. [Instalação](#2-instalação)
3. [Arquitetura](#3-arquitetura)
4. [luany/core — HTTP e Roteamento](#4-luanycore--http-e-roteamento)
5. [luany/database — ORM e Migrações](#5-luanydatabase--orm-e-migrações)
6. [luany/framework — Camada de Aplicação](#6-luanyframework--camada-de-aplicação)
7. [luany/lte — Motor de Templates](#7-luanylte--motor-de-templates)
8. [luany/cli — Ferramenta de Linha de Comando](#8-luanycli--ferramenta-de-linha-de-comando)
9. [luany/luany — Esqueleto da Aplicação](#9-luanyluany--esqueleto-da-aplicação)
10. [Referência de Configuração](#10-referência-de-configuração)
11. [Variáveis de Ambiente](#11-variáveis-de-ambiente)
12. [Referência de Comandos CLI](#12-referência-de-comandos-cli)
13. [Referência de Templates LTE](#13-referência-de-templates-lte)
14. [Guia de Integração](#14-guia-de-integração)
15. [Guia de Atualização](#15-guia-de-atualização)

---

## 1. Visão Geral dos Pacotes

| Pacote | Nome Composer | Descrição | Versão |
|--------|---------------|-----------|--------|
| **Core** | `luany/core` | HTTP Request/Response, Router, Pipeline de Middleware, CORS, Rate Limiting | v1.0.0 |
| **Database** | `luany/database` | Conexão PDO, QueryBuilder Fluente, ORM Active Record, Relações, Migrações | v1.0.0 |
| **Framework** | `luany/framework` | Container IoC, Kernel, Service Providers, Config, Session, Validator, i18n | v1.0.0 |
| **LTE** | `luany/lte` | Motor de templates compilado por AST (ficheiros `.lte`) | v1.0.0 |
| **CLI** | `luany/cli` | Ferramenta `luany`, scaffolding CRUD completo, migrações, LDE | v1.0.2 |
| **Skeleton** | `luany/luany` | Template de aplicação pronto a usar | v1.1.0 |

### Grafo de Dependências

```
luany/luany (skeleton)
├── luany/framework
│   ├── luany/core
│   └── luany/lte
├── luany/database
└── luany/cli (dev tool)
```

---

## 2. Instalação

### Novo Projeto

```bash
# Com Luany CLI (recomendado)
composer global require luany/cli
luany new my-app
cd my-app
luany doctor
luany migrate
luany dev
```

```bash
# Ou directamente via Composer
composer create-project luany/luany my-app
cd my-app
luany key:generate
luany serve
```

### Projeto Existente

```bash
composer require luany/framework luany/database
composer require --dev luany/cli
```

### Live Reload (Desenvolvimento)

```bash
npm install
luany dev
```

Inicia o servidor PHP built-in e activa o live reload via **Luany Dev Engine (LDE)** — views, assets e ficheiros PHP disparam actualizações instantâneas no browser sem camada de proxy.

**Estratégia de reload:**
- `.css` — CSS injectado ao vivo, sem reload completo
- `.lte` / `.php` / `.js` / `routes/` / `config/` — reload completo da página

> **Nota:** `luany serve` continua a funcionar como servidor PHP simples sem live reload.

---

## 3. Arquitetura

### Ciclo de Vida da Requisição

```
Browser → public/index.php
           ↓
       bootstrap/app.php
       (Env::load, Providers, Singleton bindings)
           ↓
       Kernel::boot()
       (Config, Session, CSRF, motor LTE, auto-descoberta de rotas)
           ↓
       Kernel::handle(Request)
           ↓
       Pipeline → Middleware → Router → Controller → Response
           ↓
       Response::send()
           ↓
       Kernel::terminate()
```

### Estrutura de Diretórios (Skeleton)

```
my-app/
├── app/
│   ├── Controllers/       # Controladores HTTP
│   ├── Exceptions/        # Handler personalizado
│   ├── Http/
│   │   ├── Kernel.php     # HTTP Kernel
│   │   └── Middleware/    # Middleware da aplicação (incl. DevMiddleware)
│   ├── Models/            # Modelos Active Record
│   ├── Providers/         # Service providers
│   └── Support/           # Helpers da aplicação
├── bootstrap/
│   └── app.php            # Bootstrap da aplicação
├── config/
│   ├── app.php            # Configuração da aplicação
│   └── mail.php           # Configuração de e-mail
├── database/
│   └── migrations/        # Migrações com timestamp
├── lang/                  # Ficheiros de tradução (en.php, pt.php)
├── public/
│   ├── index.php          # Front controller
│   └── assets/            # CSS, JS, imagens
├── routes/
│   ├── http.php           # Definições de rotas principais
│   └── *.php              # Ficheiros de rotas por feature (auto-descobertos)
├── storage/
│   ├── cache/views/       # Cache LTE compilada
│   └── logs/              # Ficheiros de log
├── views/
│   ├── components/        # Parciais reutilizáveis
│   ├── layouts/           # Templates de layout
│   └── pages/             # Views de páginas
├── .env                   # Configuração de ambiente
├── package.json           # Dependências Node (chokidar + ws para LDE live reload)
└── composer.json
```

---

## 4. luany/core — HTTP e Roteamento

**Namespace**: `Luany\Core`

### 4.1 Request

**Classe**: `Luany\Core\Http\Request`

Wrapper de requisição HTTP imutável.

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

**Métodos Públicos**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `fromGlobals` | `static fromGlobals(): self` | Criar a partir das superglobais PHP |
| `method` | `method(): string` | Método HTTP (GET, POST, etc.) |
| `uri` | `uri(): string` | Caminho URI da requisição |
| `header` | `header(string $name, ?string $default = null): ?string` | Obter cabeçalho (case-insensitive) |
| `body` | `body(): array` | Campos do corpo apenas (exclui query string) |
| `all` | `all(): array` | Parâmetros query + post mesclados |
| `input` | `input(string $key, mixed $default = null): mixed` | Obter um valor de input |
| `query` | `query(string $key, mixed $default = null): mixed` | Parâmetro GET |
| `post` | `post(string $key, mixed $default = null): mixed` | Parâmetro POST |
| `cookie` | `cookie(string $name, ?string $default = null): ?string` | Valor do cookie |
| `server` | `server(string $key, mixed $default = null): mixed` | Parâmetro do servidor |
| `only` | `only(array $keys): array` | Subconjunto de inputs |
| `has` | `has(string $key): bool` | Verificar se chave existe |
| `filled` | `filled(string $key): bool` | Chave existe e não está vazia |
| `isMethod` | `isMethod(string $method): bool` | Verificar método HTTP |
| `isGet` / `isPost` | `isGet(): bool` / `isPost(): bool` | Atalhos de método |
| `isAjax` | `isAjax(): bool` | Verificar cabeçalho X-Requested-With |
| `expectsJson` | `expectsJson(): bool` | Verificar Accept: application/json |
| `ip` | `ip(): string` | Endereço IP do cliente |
| `routeParams` | `routeParams(): array` | Parâmetros extraídos pelo router |
| `withRouteParams` | `withRouteParams(array $params): self` | Clonar com parâmetros de rota |

### 4.2 Response

**Classe**: `Luany\Core\Http\Response`

```php
return Response::make('<h1>Hello</h1>', 200);
return Response::json(['user' => $user]);
return Response::redirect('/dashboard');
return Response::notFound();
return Response::serverError();
```

**Métodos Públicos**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `make` | `static make(string $body = '', int $status = 200): self` | Criar resposta |
| `json` | `static json(mixed $data, int $status = 200): self` | Resposta JSON |
| `redirect` | `static redirect(string $url, int $status = 302): self` | Redireccionamento |
| `notFound` | `static notFound(): self` | Resposta 404 |
| `unauthorized` | `static unauthorized(): self` | Resposta 401 |
| `forbidden` | `static forbidden(): self` | Resposta 403 |
| `serverError` | `static serverError(): self` | Resposta 500 |
| `header` | `header(string $name, string $value): self` | Adicionar cabeçalho (fluente) |
| `withHeaders` | `withHeaders(array $headers): self` | Adicionar múltiplos cabeçalhos |
| `withCookie` | `withCookie(string $name, string $value, array $options = []): self` | Definir cookie |
| `status` | `status(int $code): self` | Definir código de estado |
| `body` | `body(string $body): self` | Definir corpo |
| `getStatusCode` | `getStatusCode(): int` | Obter código de estado |
| `getBody` | `getBody(): string` | Obter conteúdo do corpo |
| `getHeaders` | `getHeaders(): array` | Obter todos os cabeçalhos |
| `isRedirect` | `isRedirect(): bool` | É uma resposta de redireccionamento |
| `isSuccessful` | `isSuccessful(): bool` | Estado 2xx |
| `send` | `send(): void` | Enviar ao cliente |

### 4.3 Router

**Classe**: `Luany\Core\Routing\Router`

```php
use Luany\Core\Routing\Route;

Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
Route::get('/users/{id}', [UserController::class, 'show']);
Route::put('/users/{id}', [UserController::class, 'update']);
Route::delete('/users/{id}', [UserController::class, 'destroy']);

// Rotas de recurso — regista todas as 7 rotas CRUD + alias PATCH
Route::resource('products', ProductController::class);

// Grupos de rotas com prefixo e middleware
Route::group(['prefix' => '/admin', 'middleware' => [AuthMiddleware::class]], function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
});

// Rotas nomeadas
Route::get('/users/{id}', [UserController::class, 'show'])->name('users.show');

// Route model binding
Route::bind('user', fn(string $id) => User::find($id));
Route::model('user', User::class);

// Cache de rotas (produção)
Route::cache(base_path('storage/cache/routes.php'));
Route::loadCache(base_path('storage/cache/routes.php'));
```

**Métodos Estáticos de Route**:

| Método | Assinatura |
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
| `cache` | `static cache(string $path): void` | Guardar rotas compiladas |
| `loadCache` | `static loadCache(string $path): bool` | Carregar rotas em cache |
| `clearCache` | `static clearCache(string $path): void` | Eliminar cache de rotas |
| `reset` | `static reset(): void` | Reiniciar estado do router (testes) |
| `getRoutes` | `static getRoutes(): array` | Obter todas as rotas registadas |

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

**Classe**: `Luany\Core\Middleware\CorsMiddleware`

Middleware CORS configurável.

```php
$cors = new CorsMiddleware([
    'allowed_origins'   => ['https://app.example.com', '*.example.com'],
    'allowed_methods'   => ['GET', 'POST', 'PUT', 'DELETE'],
    'allowed_headers'   => ['Content-Type', 'Authorization'],
    'exposed_headers'   => [],
    'max_age'           => 86400,
    'allow_credentials' => false,
]);
```

### 4.7 Rate Limiting

**Interface**: `Luany\Core\RateLimit\RateLimiterInterface`

```php
// Em memória (testes / processo único)
$limiter = new InMemoryRateLimiter();

// Baseado em ficheiros (múltiplos processos)
$limiter = new FileRateLimiter(storage_path('rate-limits'));

$limiter->attempt('chave', maxAttempts: 60, decaySeconds: 60);
$limiter->remaining('chave', maxAttempts: 60);
$limiter->tooManyAttempts('chave', maxAttempts: 60);
$limiter->reset('chave');
```

**RateLimitMiddleware**:

```php
Route::group(['middleware' => [new RateLimitMiddleware($limiter, 60, 60)]], function () {
    Route::post('/api/login', [AuthController::class, 'login']);
});
```

### 4.8 Excepções

| Classe | Código | Descrição |
|--------|--------|-----------|
| `RouteNotFoundException` | 404 | Nenhuma rota correspondeu à requisição |
| `MethodNotAllowedException` | 405 | URI correspondeu mas o método HTTP não — inclui cabeçalho `Allow` |

---

## 5. luany/database — ORM e Migrações

**Namespace**: `Luany\Database`

### 5.1 Connection

**Classe**: `Luany\Database\Connection`

```php
// Configurar (chamado pelo DatabaseServiceProvider)
Connection::configure([
    'host'    => '127.0.0.1',
    'port'    => 3306,
    'dbname'  => 'my_app',
    'user'    => 'root',
    'pass'    => '',
    'charset' => 'utf8mb4',
]);

// Suporte a transações
$connection->beginTransaction();
$connection->commit();
$connection->rollBack();

// Callback de transação (commit/rollback automático)
$connection->transaction(function (\PDO $pdo) {
    // operações
});
```

### 5.2 QueryBuilder

**Classe**: `Luany\Database\QueryBuilder`

QueryBuilder totalmente fluente com prepared statements.

```php
$qb = new QueryBuilder($connection);

$users = $qb->table('users')
    ->select('id', 'name', 'email')
    ->where('active', '=', 1)
    ->orWhere('role', '=', 'admin')
    ->whereIn('status', ['active', 'pending'])
    ->whereNotNull('email_verified_at')
    ->orderBy('name', 'ASC')
    ->limit(10)
    ->offset(20)
    ->get();

$existe = $qb->table('users')->where('email', '=', $email)->exists();
$resultado = $qb->table('products')->paginate(perPage: 15, page: 2);
```

**Métodos Fluentes**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `table` | `table(string $table): static` | Definir tabela alvo |
| `select` | `select(string ...$columns): static` | Colunas a selecionar |
| `where` | `where(string $col, string $op, mixed $val): static` | Cláusula AND WHERE |
| `orWhere` | `orWhere(string $col, string $op, mixed $val): static` | Cláusula OR WHERE |
| `whereIn` | `whereIn(string $col, array $vals): static` | WHERE IN |
| `whereNull` | `whereNull(string $col): static` | WHERE IS NULL |
| `whereNotNull` | `whereNotNull(string $col): static` | WHERE IS NOT NULL |
| `orderBy` | `orderBy(string $col, string $dir = 'ASC'): static` | ORDER BY |
| `limit` | `limit(int $limit): static` | LIMIT |
| `offset` | `offset(int $offset): static` | OFFSET |
| `get` | `get(): array` | Executar SELECT, retornar linhas |
| `first` | `first(): ?array` | Primeira linha ou null |
| `insert` | `insert(array $data): bool` | Inserir linha |
| `update` | `update(array $data): int` | Actualizar linhas, retornar contagem |
| `delete` | `delete(): int` | Eliminar linhas, retornar contagem |
| `count` | `count(): int` | Contar linhas correspondentes |
| `exists` | `exists(): bool` | Verificar se alguma linha corresponde |
| `paginate` | `paginate(int $perPage = 15, int $page = 1): PaginationResult` | Resultados paginados |
| `raw` | `raw(string $sql, array $bindings = []): Result` | SELECT bruto |
| `query` | `query(string $sql, array $bindings = []): Result` | Alias para raw |
| `statement` | `statement(string $sql, array $bindings = []): int` | INSERT/UPDATE/DELETE bruto |
| `lastInsertId` | `lastInsertId(): string` | Último ID inserido |

### 5.3 PaginationResult

**Classe**: `Luany\Database\PaginationResult`

```php
$paginacao = User::newQuery()->paginate(15, page: 2);

$paginacao->data;        // array de linhas
$paginacao->total;       // contagem total de registos
$paginacao->perPage;     // registos por página
$paginacao->currentPage; // número da página actual
$paginacao->lastPage;    // número da última página
$paginacao->from;        // primeiro registo nesta página
$paginacao->to;          // último registo nesta página
$paginacao->hasMore();   // bool — existem mais páginas
$paginacao->hasPrev();   // bool — existe página anterior
$paginacao->toArray();   // representação em array completa
```

### 5.4 Result

**Classe**: `Luany\Database\Result`

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `fetchAll` | `fetchAll(): array` | Todas as linhas como arrays associativos |
| `fetchOne` | `fetchOne(): ?array` | Primeira linha ou null |
| `fetchColumn` | `fetchColumn(int $index = 0): array` | Valores de uma única coluna |
| `rowCount` | `rowCount(): int` | Número de linhas afectadas/retornadas |

### 5.5 Model

**Classe**: `Luany\Database\Model` (abstrata)

Classe base ORM Active Record.

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

**Utilização**:

```php
$user  = User::find(42);
$users = User::all('created_at DESC');
$admins = User::where('role = ?', ['admin']);
$first  = User::firstWhere('email = ?', [$email]);
$total  = User::count();
$page   = User::newQuery()->where('active', '=', 1)->paginate(15, 1);
$user   = User::create(['name' => 'Maria', 'email' => 'maria@example.com']);

$user = new User();
$user->name = 'João';
$user->save();

$user->update(['name' => 'João Silva']);
$user->delete();

$array = $user->toArray(); // respeita $hidden
$json  = $user->toJson();
```

**Métodos do Model**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `find` | `static find(mixed $id): ?static` | Encontrar por chave primária |
| `all` | `static all(string $orderBy = ''): array<int, static>` | Todos os registos |
| `where` | `static where(string $conditions, array $bindings = []): array<int, static>` | WHERE bruto |
| `firstWhere` | `static firstWhere(string $conditions, array $bindings = []): ?static` | Primeiro resultado |
| `count` | `static count(string $conditions = '', array $bindings = []): int` | Contagem |
| `newQuery` | `static newQuery(): QueryBuilder` | QueryBuilder limpo para a tabela |
| `create` | `static create(array $attributes): static` | Inserir e retornar |
| `save` | `save(): bool` | Persistir |
| `update` | `update(array $attributes): bool` | Actualizar este registo |
| `delete` | `delete(): bool` | Eliminar este registo |
| `fill` | `fill(array $attributes): void` | Atribuição em massa (fillable) |
| `toArray` | `toArray(): array` | Array (exclui hidden) |
| `toJson` | `toJson(): string` | JSON (exclui hidden) |
| `exists` | `exists(): bool` | Foi carregado da BD |
| `getTable` | `getTable(): string` | Nome da tabela |
| `getPrimaryKey` | `getPrimaryKey(): string` | Nome da chave primária |

### 5.6 Relações

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

**Eager loading** — previne consultas N+1:

```php
$users = User::with('posts')->all();

foreach ($users as $user) {
    foreach ($user->posts as $post) { // sem consulta extra
        echo $post->title;
    }
}
```

**Métodos de relação**:

| Método | Descrição |
|--------|-----------|
| `hasOne(related, foreignKey, localKey)` | 1:1 — tabela relacionada tem FK apontando aqui |
| `hasMany(related, foreignKey, localKey)` | 1:N — tabela relacionada tem FK apontando aqui |
| `belongsTo(related, foreignKey, ownerKey)` | Inverso — esta tabela tem FK apontando para a relacionada |
| `with(string ...$relations): static` | Estático — activa eager loading para próxima consulta |
| `getRelation(string $relation): mixed` | Obter relação em cache ou por lazy loading |
| `setRelation(string $relation, mixed $value): void` | Definir valor de relação |

### 5.7 SoftDeletes

**Trait**: `Luany\Database\Concerns\SoftDeletes`

```php
class Post extends Model
{
    use SoftDeletes;
    // Requer coluna `deleted_at TIMESTAMP NULL` na migração
}

$post->delete();         // define deleted_at, NÃO elimina fisicamente
$post->trashed();        // true se soft-deleted
$post->restore();        // limpa deleted_at
$post->forceDelete();    // elimina fisicamente

Post::withTrashed();     // inclui soft-deleted nas consultas
Post::onlyTrashed();     // apenas registos soft-deleted
```

### 5.8 Sistema de Migrações

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

---

## 6. luany/framework — Camada de Aplicação

**Namespace**: `Luany\Framework`

### 6.1 Application

**Classe**: `Luany\Framework\Application`

Container IoC. Implementa `ApplicationInterface`.

```php
$app = new Application('/path/to/project');

$app->singleton('db', fn($app) => new Connection($config));
$app->bind('logger', fn($app) => new Logger());

$db = $app->make('db');
$app->register(new AppServiceProvider());
```

### 6.2 HTTP Kernel

**Classe**: `Luany\Framework\Http\Kernel`

```php
class Kernel extends \Luany\Framework\Http\Kernel
{
    protected array $middleware = [
        DevMiddleware::class,    // PRIMEIRO — LDE live reload (zero custo em produção)
        LocaleMiddleware::class,
        CsrfMiddleware::class,
    ];
    // Sem override de $routesFile necessário — auto-descoberta carrega todos os routes/*.php
}
```

**Auto-descoberta de rotas**: O Kernel carrega `routes/http.php` primeiro, depois todos os outros ficheiros `*.php` na directoria `routes/` por ordem alfabética.

### 6.3 Config

**Classe**: `Luany\Framework\Support\Config`

```php
config('app.name');
config('app.debug');
config('app.locale', 'en');
config()->set('app.name', 'Novo Nome');
config()->has('app.debug');
config()->all();
```

### 6.4 Session

**Interface**: `Luany\Framework\Contracts\SessionInterface`
**Classe**: `Luany\Framework\Session\FileSession`

```php
session()->set('user_id', 42);
session()->get('user_id');
session()->has('user_id');
session()->forget('user_id');
session()->flash('status', 'Perfil actualizado.');
session()->regenerate();
session()->all();
session()->destroy();
```

**Ciclo de vida dos dados flash**:
- `flash('chave', $valor)` — armazenado como "new"
- Próxima requisição: "new" torna-se "old" — disponível via `get('chave')`
- Requisição seguinte: "old" é eliminado

### 6.5 Protecção CSRF

**Classe**: `Luany\Framework\Security\CsrfToken`

```php
csrf_token();
// Nos formulários: @csrf gera <input type="hidden" name="csrf_token" value="...">
// Em AJAX: cabeçalho X-CSRF-Token
```

**CsrfMiddleware** — protege POST/PUT/PATCH/DELETE. Rotas isentas via lista `$except`.

### 6.6 Validator

**Classe**: `Luany\Framework\Validation\Validator`

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
    $errors = $validator->errors();
}

$validated = $validator->validated();
```

**Regras suportadas**:

| Regra | Descrição |
|-------|-----------|
| `required` | Campo deve estar presente e não vazio |
| `string` | Deve ser uma string |
| `email` | Deve ser um e-mail válido |
| `numeric` | Deve ser numérico |
| `integer` | Deve ser um inteiro |
| `min:{n}` | Comprimento mínimo (string) ou valor mínimo (numérico) |
| `max:{n}` | Comprimento máximo (string) ou valor máximo (numérico) |
| `in:{a},{b}` | Deve ser um dos valores listados |
| `confirmed` | Deve ter `{campo}_confirmation` correspondente |
| `boolean` | Deve ser booleano |
| `unique:{tabela},{col}` | Deve ser único (baseado em callback) |

### 6.7 Exception Handler

**Classe**: `Luany\Framework\Exceptions\Handler` (abstrata)

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

### 6.8 HttpException e ValidationException

```php
abort(404);
abort(403, 'Acesso negado.');

$data = validate($request->body(), [
    'name'  => 'required|string',
    'email' => 'required|email',
], '/url-de-retorno');
```

### 6.9 Env

**Classe**: `Luany\Framework\Support\Env`

```php
Env::load('/path/to/project');
$host  = Env::get('DB_HOST', 'localhost');
$debug = Env::get('APP_DEBUG', false);
Env::required(['DB_HOST', 'DB_NAME']);
```

**Conversão de tipos**: `'true'` → `true`, `'false'` → `false`, `'null'` → `null`, `'empty'` → `''`.

### 6.10 Translator

**Classe**: `Luany\Framework\Support\Translator`

```php
__('nav.home');
__('footer.copyright', ['year' => '2026']);
```

### 6.11 Funções Helper Globais

| Função | Assinatura | Descrição |
|--------|-----------|-----------|
| `app()` | `app(?string $abstract = null): mixed` | Instância da aplicação ou resolver |
| `env()` | `env(string $key, mixed $default = null): mixed` | Variável de ambiente |
| `config()` | `config(?string $key = null, mixed $default = null): mixed` | Valor de configuração |
| `session()` | `session(): SessionInterface` | Serviço de sessão |
| `csrf_token()` | `csrf_token(): string` | Token CSRF actual |
| `old()` | `old(string $key, mixed $default = null): mixed` | Input anterior |
| `base_path()` | `base_path(string $path = ''): string` | Caminho absoluto para raiz do projeto |
| `view()` | `view(string $name, array $data = []): string` | Renderizar view LTE |
| `redirect()` | `redirect(string $url, int $status = 302): Response` | Resposta de redireccionamento |
| `response()` | `response(string $body = '', int $status = 200): Response` | Criar resposta |
| `abort()` | `abort(int $code, string $message = ''): never` | Lançar HttpException |
| `validate()` | `validate(array $data, array $rules, string $back): array` | Validar ou redirecionar |
| `__()` | `__(string $key, array $replace = []): string` | Traduzir chave |
| `locale()` | `locale(): string` | Código do locale actual |

---

## 7. luany/lte — Motor de Templates

**Namespace**: `Luany\Lte`

LTE (Luany Template Engine) compila templates `.lte` em ficheiros PHP em cache via análise AST.

### 7.1 Engine

```php
$engine = new Engine(
    viewsPath: '/path/to/views',
    cachePath: '/path/to/cache',
    autoReload: true,
);

$html = $engine->render('pages.home', ['title' => 'Bem-vindo']);

// Diretivas personalizadas
$engine->getCompiler()->directive('datetime', function (?string $args) {
    return "<?php echo date({$args}); ?>";
});
```

### 7.2 SectionStack

| Método | Descrição |
|--------|-----------|
| `reset()` | Limpar todo o estado |
| `setLayout(string)` | Definir layout pai |
| `start(string)` | Iniciar captura de secção |
| `end()` | Terminar captura de secção |
| `get(string, string): string` | Obter conteúdo da secção |
| `startPush(string)` | Iniciar push para stack nomeada |
| `endPush()` | Terminar captura de push |
| `getStack(string): string` | Renderizar stack nomeada |

### 7.3 AssetStack

| Método | Descrição |
|--------|-----------|
| `startStyle(array)` | Iniciar captura de bloco style |
| `endStyle()` | Terminar captura de style |
| `startScript(array)` | Iniciar captura de bloco script |
| `endScript()` | Terminar captura de script |
| `renderStyles(): string` | Renderizar todas as tags `<style>` |
| `renderScripts(): string` | Renderizar todas as tags `<script>` |

---

## 8. luany/cli — Ferramenta de Linha de Comando

**Namespace**: `LuanyCli`

### 8.1 Arquitetura

```
LuanyCli\Application::run($argv)
    ↓
CommandRegistry → CommandInterface → handle($args)
```

### 8.2 Classes de Suporte

| Classe | Descrição |
|--------|-----------|
| `EnvParser` | Analisar ficheiros `.env` (trata `=` base64 em valores) |
| `FieldParser` | Converter definições de campos para colunas de migração, HTML de formulário, casts, regras de validação |
| `ProjectFinder` | Percorrer árvore de directórios para encontrar o projeto Luany mais próximo |
| `StubRenderer` | Substituir `{{placeholders}}` em ficheiros stub |

### 8.3 Classes Dev (LDE)

| Classe | Descrição |
|--------|-----------|
| `Dev\ProcessManager` | Orquestra servidor PHP + watcher Node via `proc_open()`. Tick loop, shutdown limpo (SIGTERM → SIGKILL) |
| `Dev\NodeRunner` | Spawna o processo watcher Node.js. Verifica `node`, `chokidar`, `ws`, `watcher.js` |

**Recursos**:

| Ficheiro | Descrição |
|----------|-----------|
| `Resources/dev/watcher.js` | Watcher Chokidar + servidor WebSocket. Debounce 40ms. CSS → inject, PHP/LTE/JS → reload |
| `Resources/dev/client.js` | Cliente WebSocket do browser. CSS inject com cache-buster. Reconnect com back-off exponencial |

---

## 9. luany/luany — Esqueleto da Aplicação

### 9.1 Bootstrap (`bootstrap/app.php`)

```php
// Sequência:
// 1. Carregar autoloader do Composer
// 2. new Application(caminho base)
// 3. Env::load() + Env::required(['APP_ENV', 'APP_URL'])
// 4. Registar AppServiceProvider, DatabaseServiceProvider
// 5. Registar Kernel e Handler como singletons
// 6. Retornar $app
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

### 9.3 Helpers da Aplicação (`app/Support/helpers.php`)

| Função | Descrição |
|--------|-----------|
| `url(string $path)` | URL absoluta |
| `asset(string $path)` | URL de asset |
| `route(string $name, array $params)` | URL de rota nomeada |
| `flash(string $type, string $message)` | Definir mensagem flash |
| `get_flash(): ?array` | Obter mensagem flash |
| `auth_user(): ?int` | ID do utilizador actual |
| `is_authenticated(): bool` | Utilizador está autenticado |
| `e(mixed $value): string` | Escapar HTML |
| `csrf_field(): string` | HTML do input CSRF oculto |

---

## 10. Referência de Configuração

### `config/app.php`

| Chave | Padrão | Descrição |
|-------|--------|-----------|
| `name` | `env('APP_NAME', 'Luany')` | Nome da aplicação |
| `env` | `env('APP_ENV', 'production')` | Ambiente (`development` activa DevMiddleware) |
| `debug` | `env('APP_DEBUG', false)` | Modo debug |
| `url` | `env('APP_URL', 'http://localhost:8000')` | URL da aplicação |
| `locale` | `env('APP_LOCALE', 'en')` | Locale padrão |
| `fallback_locale` | `'en'` | Locale de fallback |
| `supported_locales` | `['en', 'pt']` | Locales suportados |

---

## 11. Variáveis de Ambiente

### Aplicação

| Variável | Exemplo | Obrigatório | Descrição |
|----------|---------|-------------|-----------|
| `APP_NAME` | `"My App"` | Não | Nome de exibição |
| `APP_ENV` | `development` | **Sim** | Ambiente (`development` activa DevMiddleware) |
| `APP_DEBUG` | `true` | Não | Modo debug |
| `APP_URL` | `http://localhost:8000` | **Sim** | URL base |
| `APP_KEY` | `base64:...` | Não | Chave de encriptação |
| `APP_TIMEZONE` | `UTC` | Não | Fuso horário |
| `APP_LOCALE` | `en` | Não | Locale padrão |

### Base de Dados

| Variável | Exemplo | Descrição |
|----------|---------|-----------|
| `DB_HOST` | `127.0.0.1` | Host MySQL |
| `DB_PORT` | `3306` | Porta MySQL |
| `DB_NAME` | `luany` | Nome da base de dados |
| `DB_USER` | `root` | Nome de utilizador |
| `DB_PASS` | *(vazio)* | Palavra-passe |

### E-mail (opcional)

| Variável | Exemplo |
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

## 12. Referência de Comandos CLI

### Global — executar em qualquer lugar

| Comando | Descrição |
|---------|-----------|
| `luany new <n>` | Criar um novo projeto Luany |
| `luany doctor` | Verificação completa do ambiente e saúde do projeto |
| `luany about` | Exibir informações do projeto e ambiente |
| `luany list` | Listar todos os comandos disponíveis |

### Projeto — requer um projeto Luany válido

| Comando | Descrição |
|---------|-----------|
| `luany serve` | Iniciar servidor PHP simples (`localhost:8000`) — sem live reload |
| `luany dev` | Iniciar Luany Dev Engine — servidor PHP + live reload (LDE) |
| `luany key:generate` | Gerar APP_KEY e escrever no `.env` |
| `luany cache:clear` | Limpar cache de views compiladas |
| `luany route:list` | Exibir todas as rotas registadas numa tabela |

### luany dev

```bash
luany dev
luany dev localhost 8080          # host/porto personalizado
luany dev localhost 8000 35730    # porto WebSocket personalizado
```

**Requisitos:** Node.js no PATH · `npm install` executado · `APP_ENV=development` no `.env`

**Arquitectura:**
```
Browser ←──────────────────→ PHP   (porto 8000) — directo, sem proxy
Browser ←── WebSocket ──────→ Node (porto 35729) — apenas sinais de reload
```

O DevMiddleware intercepta respostas HTML e injeta o cliente LDE antes do `</body>`. O cliente liga ao servidor WebSocket e aplica mudanças ao vivo — CSS sem reload de página, PHP/LTE com reload limpo.

### Scaffolding

| Comando | Saída |
|---------|-------|
| `luany make:controller <n>` | `app/Controllers/{Name}Controller.php` |
| `luany make:model <n>` | `app/Models/{Name}.php` |
| `luany make:migration <n>` | `database/migrations/{timestamp}_{name}.php` |
| `luany make:middleware <n>` | `app/Http/Middleware/{Name}Middleware.php` |
| `luany make:provider <n>` | `app/Providers/{Name}ServiceProvider.php` |
| `luany make:view <n> [tipo]` | `views/{path}.lte` (tipos: `page`, `component`, `layout`) |
| `luany make:request <n>` | `app/Http/Requests/{Name}Request.php` |
| `luany make:test <n>` | `tests/{Name}Test.php` |
| `luany make:feature <n> [campos...]` | Model + Controller + Migration + 4 Views + Rotas |

**`luany make:feature`** — interactivo ou inline:

```bash
# Modo interactivo
luany make:feature Product

# Modo inline (sem perguntas)
luany make:feature Product name:string price:decimal active:boolean
```

Tipos de campo suportados: `string`, `text`, `integer`, `boolean`, `email`, `date`, `decimal`.

Ficheiros gerados por feature:

```
app/Models/Product.php
app/Controllers/ProductController.php           ← CRUD completo (7 métodos)
database/migrations/{ts}_create_products_table.php
views/pages/products/index.lte
views/pages/products/show.lte
views/pages/products/create.lte
views/pages/products/edit.lte
routes/products.php                             ← Route::resource('products', ...)
```

**Mapeamento de tipos de campo**:

| Tipo | Migração | Input | Cast | Comportamento |
|------|----------|-------|------|---------------|
| `string` | `VARCHAR(255)` | `text` | — | `required`, `placeholder` |
| `text` | `TEXT` | `textarea` | — | `required`, `placeholder` |
| `integer` | `INT` | `number` | `int` | `required`, `placeholder` |
| `boolean` | `TINYINT(1)` | `checkbox` | `bool` | — |
| `email` | `VARCHAR(150)` | `email` | — | `required`, `placeholder` |
| `date` | `DATE` | `date` | — | `required`, `placeholder` |
| `decimal` | `DECIMAL(10,2)` | `number step="0.01"` | `float` | `required`, `placeholder` |

### Suporte a subdiretórios

```bash
luany make:controller Auth/Login     # → app/Controllers/Auth/LoginController.php
luany make:middleware Auth/Token     # → app/Http/Middleware/Auth/TokenMiddleware.php
```

### Migrações

| Comando | Descrição |
|---------|-----------|
| `luany migrate` | Executar todas as migrações pendentes |
| `luany migrate:rollback` | Reverter último batch |
| `luany migrate:fresh` | Eliminar todas as tabelas + re-executar todas |
| `luany migrate:status` | Mostrar tabela de estado das migrações |

---

## 13. Referência de Templates LTE

### Saída

| Sintaxe | Saída | Descrição |
|---------|-------|-----------|
| `{{ $var }}` | `htmlspecialchars($var)` | Echo com escape |
| `{!! $var !!}` | `echo $var` | Echo bruto/sem escape |
| `{{-- comment --}}` | *(removido)* | Comentário de template |

### Condicionais

```lte
@if($user)
    <p>Olá, {{ $user->name }}</p>
@elseif($guest)
    <p>Bem-vindo, visitante</p>
@else
    <p>Por favor faça login</p>
@endif

@unless($banido)   <p>Bem-vindo!</p>   @endunless
@isset($title)     <h1>{{ $title }}</h1>   @endisset
@ifempty($items)   <p>Sem itens.</p>   @endifempty
```

### Loops

```lte
@foreach($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

@forelse($users as $user)
    <li>{{ $user->name }}</li>
@empty
    <p>Nenhum utilizador encontrado.</p>
@endforelse

@for($i = 0; $i < 10; $i++)
    <span>{{ $i }}</span>
@endfor
```

### Sistema de Layout

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

**Página filha**:

```lte
@extends('layouts.main')

@section('title')A Minha Página@endsection

@push('head')
    <meta name="description" content="...">
@endpush

@section('content')
    <h1>Bem-vindo</h1>
@endsection
```

### Includes

```lte
@include('components.navbar')
@include('components.card', ['title' => 'Olá'])
```

### Assets (Estilos e Scripts)

```lte
@style
    .card { padding: 1rem; background: var(--color-bg-card); }
@endstyle

@script(defer)
    document.querySelector('.card').addEventListener('click', fn);
@endscript
```

### Segurança

```lte
<form method="POST">
    @csrf
    @method('PUT')
    ...
</form>
```

### Novidades na v1.0

```lte
<script>
    const data = @json($items);
</script>

<div @class(['active' => $isActive, 'disabled' => $isDisabled, 'card' => true])>
    ...
</div>
```

### Guardas de Autenticação

```lte
@auth
    <p>Bem-vindo, utilizador autenticado!</p>
@endauth

@guest
    <a href="/login">Entrar</a>
@endguest
```

### Debug

```lte
@dump($variable)
@dd($variable)
```

---

## 14. Guia de Integração

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

Registar em `bootstrap/app.php`:

```php
$app->register(new App\Providers\CacheServiceProvider());
```

### Middleware Personalizado

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

### Padrão de Validação de Formulários

```php
public function store(Request $request): Response
{
    $data = validate($request->body(), [
        'name'  => 'required|string|max:255',
        'email' => 'required|email',
        'price' => 'required|numeric',
    ], '/products/create');

    Product::create($data);
    flash('success', 'Produto criado com sucesso.');
    return redirect('/products');
}
```

Na view:

```lte
@if(session()->get('errors'))
    @foreach(session()->get('errors') as $field => $messages)
        <p class="error">{{ $messages[0] }}</p>
    @endforeach
@endif

<input type="text" name="name" value="{{ old('name') }}">
```

---

## 15. Guia de Atualização

### v0.x → v1.0

Consulte o `UPGRADE.md` no repositório do skeleton para o guia completo.

**Resumo das alterações incompatíveis**:

| Área | Alteração | Acção |
|------|-----------|-------|
| `abort()` | Movido para `luany/framework` — lança `HttpException` | Remover helper personalizado |
| `csrf_token()` | Chave de sessão corrigida (`csrf_token`) | Remover override personalizado |
| Bootstrap da sessão | `Kernel` gere a sessão — sem `session_start()` manual | Remover do `AppServiceProvider` |
| Ficheiros de rota | Auto-descoberta activa (`routes/*.php`) | Nenhuma acção necessária |
| `make:feature` | Gera `routes/{slug}.php` em vez de anexar ao `http.php` | Nenhuma acção necessária |
| Requisito PHP | `>=8.1` elevado para `>=8.2` | Actualizar servidor/CI |

### v1.0 → v1.1 (apenas skeleton)

| Área | Alteração | Acção |
|------|-----------|-------|
| `npm run dev` | Substituído por `luany dev` (LDE) | Executar `npm install`, usar `luany dev` |
| `DevMiddleware` | Adicionado ao skeleton — registar como primeiro middleware | Adicionar `DevMiddleware::class` primeiro em `Kernel::$middleware` |
| `package.json` | BrowserSync substituído por `chokidar` + `ws` | Executar `npm install` após actualização |

**Actualizar pacotes**:

```bash
composer update luany/core luany/framework luany/database luany/lte
composer update luany/cli
```

---

*Última actualização: 2026-03-28 — luany/cli v1.0.2 · luany/luany v1.1.0*