# Luany Ecosystem — Documentação Completa

> **Luany** — Framework PHP MVC compilado por AST com ciclo de vida de requisição explícito e motor de templates sem regex.

**Versão**: 0.3.x &nbsp;|&nbsp; **PHP**: ≥ 8.1 &nbsp;|&nbsp; **Licença**: MIT
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
| **Core** | `luany/core` | HTTP Request/Response, Router, Pipeline de Middleware | v0.2.4 |
| **Database** | `luany/database` | Conexão PDO, Query Builder, Model, Migrações | v0.1.3 |
| **Framework** | `luany/framework` | Container da aplicação, Kernel, Service Providers, i18n | v0.3.1 |
| **LTE** | `luany/lte` | Motor de templates compilado por AST (ficheiros `.lte`) | v0.2.x |
| **CLI** | `luany/cli` | Ferramenta de linha de comando `luany`, scaffolding, migrações | v0.2.2 |
| **Skeleton** | `luany/luany` | Template de aplicação pronto a usar | — |

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

### CLI Global

```bash
composer global require luany/cli
luany new my-app
```

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

### Estrutura de Diretórios (Skeleton)

```
my-app/
├── app/
│   ├── Controllers/       # Controladores HTTP
│   ├── Exceptions/        # Handler personalizado
│   ├── Http/
│   │   ├── Kernel.php     # HTTP Kernel
│   │   └── Middleware/     # Middleware da aplicação
│   ├── Models/            # Modelos estilo Eloquent
│   ├── Providers/         # Service providers
│   └── Support/           # Helpers
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
│   └── http.php           # Definições de rotas
├── storage/
│   ├── cache/views/       # Cache LTE compilada
│   └── logs/              # Ficheiros de log
├── views/
│   ├── components/        # Parciais reutilizáveis
│   ├── layouts/           # Templates de layout
│   └── pages/             # Views de páginas
├── .env                   # Configuração de ambiente
└── composer.json
```

---

## 4. luany/core — HTTP e Roteamento

**Namespace**: `Luany\Core`

### 4.1 Request

**Classe**: `Luany\Core\Http\Request`

Wrapper HTTP de requisição imutável. Criado via `Request::fromGlobals()` ou construção manual.

```php
// A partir das superglobais (index.php)
$request = Request::fromGlobals();

// Construção manual (testes)
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
| `body` | `body(): string` | Corpo bruto da requisição |
| `all` | `all(): array` | Parâmetros query + post mesclados |
| `input` | `input(string $key, mixed $default = null): mixed` | Obter um valor de input |
| `query` | `query(string $key, mixed $default = null): mixed` | Parâmetro GET |
| `post` | `post(string $key, mixed $default = null): mixed` | Parâmetro POST |
| `cookie` | `cookie(string $name, ?string $default = null): ?string` | Valor do cookie |
| `server` | `server(string $key, mixed $default = null): mixed` | Parâmetro do servidor |
| `only` | `only(array $keys): array` | Subconjunto de inputs |
| `has` | `has(string $key): bool` | Verificar se a chave existe |
| `isMethod` | `isMethod(string $method): bool` | Verificar método HTTP |
| `isGet` / `isPost` | `isGet(): bool` / `isPost(): bool` | Atalhos de método |
| `routeParams` | `routeParams(): array` | Parâmetros extraídos pelo router |
| `withRouteParams` | `withRouteParams(array $params): self` | Clonar com parâmetros de rota |

### 4.2 Response

**Classe**: `Luany\Core\Http\Response`

Construtor de resposta HTTP com métodos estáticos de fábrica.

```php
// Básico
return Response::make('<h1>Hello</h1>', 200);

// JSON
return Response::json(['user' => $user]);

// Redireccionamento
return Response::redirect('/dashboard');

// Respostas de erro
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
| `serverError` | `static serverError(): self` | Resposta 500 |
| `header` | `header(string $name, string $value): self` | Adicionar cabeçalho |
| `withCookie` | `withCookie(string $name, string $value, array $options = []): self` | Definir cookie |
| `getStatusCode` | `getStatusCode(): int` | Obter código de estado |
| `getBody` | `getBody(): string` | Obter conteúdo do corpo |
| `getHeaders` | `getHeaders(): array` | Obter todos os cabeçalhos |
| `send` | `send(): void` | Enviar ao cliente (cabeçalhos + corpo) |

### 4.3 Router

**Classe**: `Luany\Core\Routing\Router`

Recolhe definições de rotas e despacha requisições.

```php
use Luany\Core\Routing\Route;

Route::get('/users', [UserController::class, 'index']);
Route::post('/users', [UserController::class, 'store']);
Route::get('/users/{id}', [UserController::class, 'show']);
Route::put('/users/{id}', [UserController::class, 'update']);
Route::delete('/users/{id}', [UserController::class, 'destroy']);

// Rotas de recurso (todas as 7 rotas CRUD)
Route::resource('products', ProductController::class);

// Grupos de rotas
Route::group(['prefix' => '/admin', 'middleware' => [AuthMiddleware::class]], function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);
});
```

**Métodos Estáticos de Route** (em `Luany\Core\Routing\Route`):

| Método | Assinatura |
|--------|-----------|
| `get` | `static get(string $uri, array\|callable $action): void` |
| `post` | `static post(string $uri, array\|callable $action): void` |
| `put` | `static put(string $uri, array\|callable $action): void` |
| `patch` | `static patch(string $uri, array\|callable $action): void` |
| `delete` | `static delete(string $uri, array\|callable $action): void` |
| `resource` | `static resource(string $name, string $controller): void` |
| `group` | `static group(array $attributes, callable $callback): void` |

**Métodos do Router**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `handle` | `handle(Request $request): Response` | Despachar requisição para a rota correspondente |
| `addRoute` | `addRoute(string $method, string $uri, array\|callable $action): void` | Registar uma rota |

**Parâmetros de Rota**: Parâmetros em `{chaves}` são extraídos e injetados nos métodos do controlador.

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

**Classe**: `Luany\Core\Middleware\Pipeline`

Executa middleware em sequência, envolvendo o handler final.

```php
$pipeline = new Pipeline();
$response = $pipeline
    ->send($request)
    ->through([AuthMiddleware::class, LogMiddleware::class])
    ->then(function (Request $request) {
        return $router->handle($request);
    });
```

**Métodos**:

| Método | Assinatura |
|--------|-----------|
| `send` | `send(Request $request): self` |
| `through` | `through(array $middleware): self` |
| `then` | `then(callable $destination): Response` |

### 4.6 RouteNotFoundException

**Classe**: `Luany\Core\Routing\RouteNotFoundException`

Lançada quando nenhuma rota corresponde à requisição. Estende `\RuntimeException`.

---

## 5. luany/database — ORM e Migrações

**Namespace**: `Luany\Database`

### 5.1 Connection

**Classe**: `Luany\Database\Connection`

Wrapper PDO com padrão singleton.

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

**Métodos**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `configure` | `static configure(array $config): void` | Definir configuração de conexão |
| `getInstance` | `static getInstance(): \PDO` | Obter/criar singleton PDO |
| `disconnect` | `static disconnect(): void` | Fechar conexão |

### 5.2 QueryBuilder

**Classe**: `Luany\Database\QueryBuilder`

Construtor de consultas fluente.

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

**Métodos**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `table` | `table(string $table): self` | Definir tabela alvo |
| `select` | `select(string ...$columns): self` | Colunas a selecionar |
| `where` | `where(string $column, string $operator, mixed $value): self` | Cláusula WHERE |
| `orderBy` | `orderBy(string $column, string $direction = 'ASC'): self` | ORDER BY |
| `limit` | `limit(int $limit): self` | LIMIT |
| `offset` | `offset(int $offset): self` | OFFSET |
| `get` | `get(): array` | Executar SELECT, retornar linhas |
| `first` | `first(): ?array` | Primeira linha ou null |
| `insert` | `insert(array $data): bool` | Inserir linha |
| `update` | `update(array $data): int` | Atualizar linhas, retornar contagem |
| `delete` | `delete(): int` | Eliminar linhas, retornar contagem |
| `count` | `count(): int` | Contar linhas correspondentes |
| `raw` | `raw(string $sql, array $bindings = []): Result` | Consulta bruta |

### 5.3 Result

**Classe**: `Luany\Database\Result`

Envolve PDOStatement para acesso conveniente a resultados. Implementa `Countable` e `IteratorAggregate`.

| Método | Assinatura |
|--------|-----------|
| `all` | `all(): array` |
| `first` | `first(): ?array` |
| `count` | `count(): int` |
| `column` | `column(int $index = 0): array` |
| `lastInsertId` | `lastInsertId(): string` |

### 5.4 Model

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
}
```

**Utilização**:

```php
// Encontrar por ID
$user = User::find(42);

// Todos os registos (com ordenação opcional)
$users = User::all('created_at DESC');

// Criar
$user = User::create([
    'name'  => 'Maria',
    'email' => 'maria@example.com',
]);

// Atualizar
$user->update(['name' => 'Maria Silva']);

// Eliminar
$user->delete();

// Aceder a atributos
echo $user->name;
echo $user->email;

// Serialização (oculta campos $hidden)
$array = $user->toArray();
$json  = $user->toJson();
```

**Métodos do Model**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `find` | `static find(mixed $id): ?static` | Encontrar por chave primária |
| `all` | `static all(string $orderBy = ''): array` | Todos os registos (ordem validada) |
| `create` | `static create(array $attributes): static` | Inserir e retornar instância |
| `update` | `update(array $attributes): bool` | Atualizar este registo |
| `delete` | `delete(): bool` | Eliminar este registo |
| `toArray` | `toArray(): array` | Converter para array (exclui hidden) |
| `toJson` | `toJson(): string` | Converter para JSON |
| `fill` | `fill(array $attributes): void` | Atribuição em massa de atributos fillable |

### 5.5 Sistema de Migrações

#### Migration (abstrata)

**Classe**: `Luany\Database\Migration\Migration`

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

**Classe**: `Luany\Database\Migration\MigrationRepository`

Gere a tabela de rastreamento `_migrations`. Cria-a automaticamente na primeira utilização.

| Método | Descrição |
|--------|-----------|
| `ensureTable(): void` | Criar tabela `_migrations` se não existir |
| `getRan(): array` | Lista de migrações já executadas |
| `log(name, batch): void` | Registar uma migração como executada |
| `removeLast(): array` | Obter + remover último batch |
| `status(): array` | Todas as migrações com estado ran/batch |

#### MigrationRunner

**Classe**: `Luany\Database\Migration\MigrationRunner`

Orquestra a execução, reversão e inspeção de migrações.

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `run` | `run(?callable $callback = null): int` | Executar migrações pendentes |
| `rollback` | `rollback(?callable $callback = null): int` | Reverter último batch |
| `status` | `status(): array` | Obter estado de todas as migrações |
| `dropAll` | `dropAll(\PDO $pdo): void` | Eliminar todas as tabelas |

---

## 6. luany/framework — Camada de Aplicação

**Namespace**: `Luany\Framework`

### 6.1 Application

**Classe**: `Luany\Framework\Application`

Container IoC e kernel da aplicação. Implementa `ApplicationInterface`.

```php
$app = new Application('/path/to/project');

// Registar um singleton
$app->singleton('db', fn($app) => new Connection($config));

// Registar uma factory (nova instância a cada chamada)
$app->bind('logger', fn($app) => new Logger());

// Resolver
$db = $app->make('db');

// Registar provider
$app->register(new AppServiceProvider());
```

**Métodos**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `__construct` | `__construct(string $basePath)` | Criar aplicação |
| `basePath` | `basePath(string $path = ''): string` | Caminho absoluto para a raiz do projeto |
| `bind` | `bind(string $abstract, callable $factory): void` | Registar binding factory |
| `singleton` | `singleton(string $abstract, callable $factory): void` | Registar singleton |
| `make` | `make(string $abstract): mixed` | Resolver binding |
| `has` | `has(string $abstract): bool` | Verificar se binding existe |
| `register` | `register(ServiceProviderInterface $provider): void` | Registar service provider |
| `boot` | `boot(): void` | Iniciar todos os providers registados |
| `getInstance` | `static getInstance(): self` | Obter instância singleton |

### 6.2 Contratos (Interfaces)

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

**Classe**: `Luany\Framework\ServiceProvider` (abstrata)

Classe base de conveniência com implementações padrão vazias.

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
        // Configuração pós-registo
    }
}
```

### 6.4 HTTP Kernel

**Classe**: `Luany\Framework\Http\Kernel`

Implementa `KernelInterface`. Gere o ciclo de vida da requisição com pipeline de middleware.

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

**Tratamento de erros**: Utiliza um try/catch híbrido — catch interno para exceções de rota (preservando a fase "after" do middleware), catch externo para exceções de middleware. Ambos fluem pelo exception Handler.

### 6.5 Exception Handler

**Classe**: `Luany\Framework\Exceptions\Handler` (abstrata)

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

**Métodos**:

| Método | Descrição |
|--------|-----------|
| `__construct(bool $debug = false)` | Ativar/desativar modo debug |
| `report(\Throwable $e): void` | Registar a exceção (padrão: `error_log`) |
| `render(\Throwable $e): Response` | Renderizar exceção como resposta HTTP |

No **modo debug**, renderiza uma página HTML detalhada de erro com classe, mensagem, ficheiro, linha, trace, URI e método.

### 6.6 LocaleMiddleware

**Classe**: `Luany\Framework\Http\Middleware\LocaleMiddleware`

Deteta o locale em cada requisição e define-o no Translator.

**Ordem de deteção** (da mais alta para a mais baixa prioridade):
1. Cookie `app_locale` — preferência explícita do utilizador
2. Cabeçalho HTTP `Accept-Language` — preferência do browser
3. Variável de ambiente `APP_LOCALE` — padrão da aplicação
4. `'en'` — fallback codificado

### 6.7 Env

**Classe**: `Luany\Framework\Support\Env`

Wrapper em torno do `vlucas/phpdotenv`. Converte valores string para tipos PHP.

```php
Env::load('/path/to/project');

$host  = Env::get('DB_HOST', 'localhost');
$debug = Env::get('APP_DEBUG', false); // returns boolean true/false

Env::required(['DB_HOST', 'DB_NAME']); // throws RuntimeException if missing
```

**Conversão de tipos**: `'true'` → `true`, `'false'` → `false`, `'null'` → `null`, `'empty'` → `''`.

### 6.8 Translator

**Classe**: `Luany\Framework\Support\Translator`

Motor de i18n leve. Os ficheiros de tradução ficam em `lang/{locale}.php`.

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

**Ficheiro de tradução** (`lang/en.php`):

```php
return [
    'nav.home'         => 'Home',
    'footer.copyright' => '© :year Luany',
];
```

### 6.9 Funções Helper Globais

Definidas em `Luany\Framework\Support\helpers.php`, carregadas automaticamente:

| Função | Assinatura | Descrição |
|--------|-----------|-----------|
| `app()` | `app(?string $abstract = null): mixed` | Instância da aplicação ou resolver binding |
| `env()` | `env(string $key, mixed $default = null): mixed` | Variável de ambiente |
| `base_path()` | `base_path(string $path = ''): string` | Caminho absoluto para a raiz do projeto |
| `view()` | `view(string $name, array $data = []): string` | Renderizar view LTE |
| `redirect()` | `redirect(string $url, int $status = 302): Response` | Resposta de redireccionamento |
| `response()` | `response(string $body = '', int $status = 200): Response` | Criar resposta |
| `__()` | `__(string $key, array $replace = []): string` | Traduzir chave |
| `locale()` | `locale(): string` | Código do locale atual |

---

## 7. luany/lte — Motor de Templates

**Namespace**: `Luany\Lte`

LTE (Luany Template Engine) compila templates `.lte` em ficheiros PHP em cache via análise AST.

### 7.1 Engine

**Classe**: `Luany\Lte\Engine`

Orquestra o pipeline compilar → cache → avaliar.

```php
$engine = new Engine(
    viewsPath: '/path/to/views',
    cachePath: '/path/to/cache',  // default: sys_get_temp_dir()/lte_cache
    autoReload: true,              // always recompile (dev mode)
);

$html = $engine->render('pages.home', ['title' => 'Welcome']);
```

**Fluxo de renderização**:
1. `SectionStack::reset()` + `AssetStack::reset()`
2. Compilar + avaliar view filha (popula secções/assets, define layout)
3. Se `@extends` foi usado → compilar + avaliar layout (consome secções)
4. Retornar HTML final

**Métodos**:

| Método | Assinatura | Descrição |
|--------|-----------|-----------|
| `render` | `render(string $view, array $data = []): string` | Renderizar view para HTML |
| `getCompiler` | `getCompiler(): Compiler` | Aceder ao compilador (diretivas personalizadas) |
| `clearCache` | `clearCache(): void` | Eliminar todos os ficheiros em cache |
| `findView` | `findView(string $view): ?string` | Resolver nome com pontos para caminho de ficheiro |
| `setAutoReload` | `setAutoReload(bool $flag): void` | Alternar recompilação automática |

### 7.2 Parser

**Classe**: `Luany\Lte\Parser`

Converte código-fonte `.lte` numa AST (array de nós).

**Tipos de token**:
- `{{ $var }}` → nó `echo` (com escape HTML)
- `{!! $var !!}` → nó `raw_echo` (sem escape)
- `@directive(args)` → nó `directive`
- `{{-- comment --}}` → removido (não está na AST)
- `@php ... @endphp` → nó `php_block`
- Texto simples → nó `text`

### 7.3 Compiler

**Classe**: `Luany\Lte\Compiler`

Compila nós AST em strings PHP executáveis.

**Diretivas personalizadas**:

```php
$engine->getCompiler()->directive('datetime', function (?string $args) {
    return "<?php echo date({$args}); ?>";
});
```

### 7.4 SectionStack

**Classe**: `Luany\Lte\SectionStack`

Registo estático para secções de layout e stacks.

| Método | Descrição |
|--------|-----------|
| `reset()` | Limpar todo o estado |
| `setLayout(string)` | Definir layout pai |
| `getLayout(): ?string` | Obter layout pai |
| `start(string)` | Iniciar captura de secção em bloco |
| `end()` | Terminar captura de secção em bloco |
| `set(string, string)` | Definir secção inline |
| `get(string, string): string` | Obter conteúdo da secção |
| `has(string): bool` | Verificar se a secção existe |
| `startPush(string)` | Iniciar push para stack nomeada |
| `endPush()` | Terminar captura de push |
| `getStack(string): string` | Renderizar stack nomeada |

### 7.5 AssetStack

**Classe**: `Luany\Lte\AssetStack`

Gere blocos inline `<style>` e `<script>` com deduplicação automática.

| Método | Descrição |
|--------|-----------|
| `reset()` | Limpar todo o estado |
| `startStyle(array)` | Iniciar captura de bloco style |
| `endStyle()` | Terminar captura de style |
| `startScript(array)` | Iniciar captura de bloco script |
| `endScript()` | Terminar captura de script |
| `renderStyles(): string` | Renderizar todas as tags `<style>` |
| `renderScripts(): string` | Renderizar todas as tags `<script>` |
| `getStyleCount(): int` | Número de styles únicos |
| `getScriptCount(): int` | Número de scripts únicos |

---

## 8. luany/cli — Ferramenta de Linha de Comando

**Namespace**: `LuanyCli`

### 8.1 Arquitetura

```
LuanyCli\Application::run($argv)
    ↓
CommandRegistry → CommandInterface → handle($args)
```

Todos os comandos estendem `BaseCommand` que implementa `CommandInterface`.

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

### 8.3 Classes de Suporte

| Classe | Descrição |
|--------|-----------|
| `EnvParser` | Analisar ficheiros `.env` manualmente (trata `=` base64 em valores) |
| `FieldParser` | Converter definições de campos para colunas de migração, HTML de formulário, etc. |
| `ProjectFinder` | Percorrer árvore de diretórios para encontrar o projeto Luany mais próximo |
| `StubRenderer` | Substituir `{{placeholders}}` em ficheiros de template stub |

---

## 9. luany/luany — Esqueleto da Aplicação

### 9.1 Bootstrap (`bootstrap/app.php`)

Sequência:
1. Carregar autoloader do Composer
2. Criar instância de `Application` com caminho base
3. `Env::load()` + `Env::required(['APP_ENV', 'APP_URL'])`
4. Registar `AppServiceProvider` e `DatabaseServiceProvider`
5. Registar `Kernel` e `Handler` personalizados como singletons
6. Retornar `$app`

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

### 9.3 Providers Padrão

**AppServiceProvider** — Regista o motor LTE (`view`) e o Translator (`translator`) no container.

**DatabaseServiceProvider** — Configura `Connection::configure()` a partir das credenciais DB do `.env`.

---

## 10. Referência de Configuração

### `config/app.php`

| Chave | Padrão | Descrição |
|-------|--------|-----------|
| `name` | `env('APP_NAME', 'Luany')` | Nome da aplicação |
| `env` | `env('APP_ENV', 'production')` | Ambiente (development/production) |
| `debug` | `env('APP_DEBUG', false)` | Ativar modo debug |
| `url` | `env('APP_URL', 'http://localhost:8000')` | URL da aplicação |
| `locale` | `env('APP_LOCALE', 'en')` | Locale padrão |
| `fallback_locale` | `'en'` | Fallback quando chave em falta |
| `supported_locales` | `['en', 'pt']` | Códigos de locale aceites |

### Adicionar um Idioma

1. Criar `lang/{code}.php` retornando um array associativo plano
2. Adicionar o código a `config/app.php` → `supported_locales`
3. Adicionar um botão de troca em `views/components/navbar.lte`

---

## 11. Variáveis de Ambiente

### Aplicação

| Variável | Exemplo | Obrigatório | Descrição |
|----------|---------|-------------|-----------|
| `APP_NAME` | `"My App"` | Não | Nome de exibição |
| `APP_ENV` | `development` | **Sim** | Ambiente |
| `APP_DEBUG` | `true` | Não | Modo debug |
| `APP_URL` | `http://localhost:8000` | **Sim** | URL base |
| `APP_KEY` | `base64:...` | Não | Chave de encriptação |
| `APP_TIMEZONE` | `UTC` | Não | Fuso horário |
| `APP_LOCALE` | `en` | Não | Locale padrão |
| `APP_HTTPS` | `false` | Não | Forçar HTTPS |

### Base de Dados

| Variável | Exemplo | Obrigatório | Descrição |
|----------|---------|-------------|-----------|
| `DB_CONNECTION` | `mysql` | Não | Driver |
| `DB_HOST` | `127.0.0.1` | Não | Host |
| `DB_PORT` | `3306` | Não | Porta |
| `DB_NAME` | `luany` | Não | Nome da base de dados |
| `DB_USER` | `root` | Não | Nome de utilizador |
| `DB_PASS` | *(vazio)* | Não | Palavra-passe |

### E-mail (opcional)

| Variável | Exemplo | Descrição |
|----------|---------|-----------|
| `MAIL_ENABLED` | `false` | Ativar e-mail |
| `MAIL_HOST` | `smtp.gmail.com` | Host SMTP |
| `MAIL_PORT` | `587` | Porta SMTP |
| `MAIL_ENCRYPTION` | `tls` | Encriptação |
| `MAIL_USERNAME` | | Utilizador SMTP |
| `MAIL_PASSWORD` | | Palavra-passe SMTP |
| `MAIL_FROM_EMAIL` | | E-mail do remetente |
| `MAIL_FROM_NAME` | `${APP_NAME}` | Nome do remetente |

---

## 12. Referência de Comandos CLI

### Projeto

| Comando | Descrição |
|---------|-----------|
| `luany new <name>` | Criar um novo projeto Luany |
| `luany serve [host] [port]` | Iniciar servidor de desenvolvimento PHP (padrão: `localhost:8000`) |
| `luany key:generate` | Gerar APP_KEY e escrever no `.env` |
| `luany cache:clear` | Limpar cache de views compiladas |
| `luany about` | Exibir informações do projeto e ambiente |
| `luany doctor` | Verificação completa de saúde (PHP, extensões, BD, ficheiros) |
| `luany list` | Listar todos os comandos disponíveis |

### Scaffolding

| Comando | Descrição | Saída |
|---------|-----------|-------|
| `luany make:controller <Name>` | Criar controlador | `app/Controllers/{Name}Controller.php` |
| `luany make:model <Name>` | Criar modelo | `app/Models/{Name}.php` |
| `luany make:migration <name>` | Criar migração | `database/migrations/{timestamp}_{name}.php` |
| `luany make:middleware <Name>` | Criar middleware | `app/Http/Middleware/{Name}Middleware.php` |
| `luany make:provider <Name>` | Criar provider | `app/Providers/{Name}ServiceProvider.php` |
| `luany make:view <name> [type]` | Criar view LTE | `views/{path}.lte` (tipos: page, component, layout) |
| `luany make:feature <Name> [fields...]` | Scaffold CRUD completo | Model + Controller + Migration + 4 Views + Routes |

**Scaffolding de feature com campos inline**:

```bash
luany make:feature Product name:string price:decimal description:text active:boolean
```

Tipos de campo suportados: `string`, `text`, `integer`, `boolean`, `email`, `date`, `decimal`.

### Migrações

| Comando | Descrição |
|---------|-----------|
| `luany migrate` | Executar todas as migrações pendentes |
| `luany migrate:rollback` | Reverter último batch |
| `luany migrate:fresh` | Eliminar todas as tabelas + re-executar todas as migrações |
| `luany migrate:status` | Mostrar tabela de estado das migrações |

### Suporte a Subdiretórios

Controladores e middleware suportam caminhos de subdiretórios:

```bash
luany make:controller Auth/Login
# → app/Controllers/Auth/LoginController.php
# → namespace App\Controllers\Auth

luany make:middleware Auth/Token
# → app/Http/Middleware/Auth/TokenMiddleware.php
```

---

## 13. Referência de Templates LTE

### Saída

| Sintaxe | Saída Compilada | Descrição |
|---------|-----------------|-----------|
| `{{ $var }}` | `htmlspecialchars($var)` | Echo com escape |
| `{!! $var !!}` | `echo $var` | Echo bruto/sem escape |
| `{{-- comment --}}` | *(removido)* | Comentário de template |

### Condicionais

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

### Sistema de Layout

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

**Página filha** (`views/pages/home.lte`):

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

As variáveis do pai são automaticamente passadas. Dados extra podem ser mesclados via o segundo argumento.

### Assets (Estilos e Scripts)

```html
@style
    .card { padding: 1rem; }
@endstyle

@script(defer)
    document.querySelector('.card').addEventListener('click', fn);
@endscript
```

Coloque `@styles` no `<head>` e `@scripts` antes de `</body>` no seu layout. Blocos duplicados (mesmo conteúdo) são automaticamente deduplicados.

### Stacks

```html
{{-- In child views --}}
@push('head')
    <link rel="stylesheet" href="/css/page.css">
@endpush

{{-- In layout --}}
@stack('head')
```

Múltiplas chamadas `@push` para a mesma stack são acumuladas (nunca substituídas).

### PHP

```html
{{-- Inline --}}
@php($count = count($items))

{{-- Block --}}
@php
    $total = array_sum(array_column($items, 'price'));
@endphp
```

### Segurança

```html
<form method="POST">
    @csrf
    @method('PUT')
    ...
</form>
```

- `@csrf` → gera input hidden com `$_SESSION['csrf_token']`
- `@method('PUT')` → gera input hidden `_method`

### Guardas de Autenticação

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

## 14. Guia de Integração

### Criar um Service Provider

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

Registar em `bootstrap/app.php`:

```php
$app->register(new App\Providers\CacheServiceProvider());
```

### Criar Middleware Personalizado

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

Adicionar a `app/Http/Kernel.php`:

```php
protected array $middleware = [
    AuthMiddleware::class,
];
```

### Diretivas LTE Personalizadas

```php
// In a service provider boot() method
$engine = app('view');
$engine->getCompiler()->directive('datetime', function (?string $args) {
    $format = $args ?: "'Y-m-d H:i'";
    return "<?php echo date({$format}); ?>";
});
```

Utilização: `@datetime('d/m/Y')`

### Utilizar o Query Builder Diretamente

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

## 15. Guia de Atualização

### v0.2.x → v0.3.x (Framework)

**Breaking**: `Kernel::handle()` agora utiliza um try/catch híbrido para execução correta da fase "after" do middleware em exceções de rota.

**Ação**: Se você sobrescreve `Kernel::handle()`, atualize a sua lógica de tratamento de erros.

### Correções de Segurança Aplicadas (v0.1.3 Database, v0.2.4 Core, v0.3.1 Framework)

1. **Injeção SQL em `Model::all()`** — `$orderBy` agora validado contra whitelist de regex estrita
2. **Poluição de `$_GET`** — Router já não escreve parâmetros de rota em `$_GET`
3. **Exceções de Middleware Não Capturadas** — Kernel captura exceções tanto de rota como de middleware

**Ação**: Atualizar os três pacotes:

```bash
composer update luany/core luany/framework luany/database
```

---

*Gerado para o ecossistema Luany na organização GitHub luany-ecosystem.*
*Última atualização: 2026-03-19*
