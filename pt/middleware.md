# Middleware

O middleware envolve o tratamento de um pedido: pode inspecionar ou alterar o pedido, interrompê-lo com a sua própria resposta ou modificar a resposta à saída. O Luany executa **middleware global** em todos os pedidos e **middleware de rota** nas rotas correspondentes, e aceita também middleware PSR-15.

## Escrever middleware

Um middleware implementa `Luany\Core\Middleware\MiddlewareInterface`:

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
            return redirect('/login');          // termina aqui
        }

        return $next($request);                 // continua
    }
}
```

O trabalho feito antes de `$next()` é executado à entrada; o trabalho feito depois é executado à saída:

```php
public function handle(Request $request, callable $next): Response
{
    $start = microtime(true);

    $response = $next($request);

    return $response->header('X-Response-Time', round((microtime(true) - $start) * 1000) . 'ms');
}
```

O middleware referenciado pelo nome da classe é construído através do [container](container.md), pelo que as dependências do construtor são injetadas:

```php
public function __construct(private LoggerInterface $logger) {}
```

## Middleware global

O middleware global é executado em todos os pedidos, antes do routing — incluindo pedidos que terminam num 404. Liste-o em `app/Http/Kernel.php`, pela ordem de execução:

```php
namespace App\Http;

use App\Http\Middleware\CsrfMiddleware;
use Luany\Framework\Http\Kernel as BaseKernel;
use Luany\Framework\Http\Middleware\LiveReloadMiddleware;
use Luany\Framework\Http\Middleware\LocaleMiddleware;

class Kernel extends BaseKernel
{
    protected array $middleware = [
        LiveReloadMiddleware::class,   // primeiro, para ver o HTML final
        CsrfMiddleware::class,
        LocaleMiddleware::class,
    ];
}
```

O skeleton regista o `LiveReloadMiddleware` (ativo apenas em pedidos locais com `APP_ENV=development`) e o [`CsrfMiddleware`](csrf.md).

## Middleware de rota

Associe middleware a uma única rota ou a um [grupo](routing.md#grupos-de-rotas):

```php
Route::get('/dashboard', [DashboardController::class, 'index'])
    ->middleware(AuthMiddleware::class);

Route::post('/posts', [PostController::class, 'store'])
    ->middleware([AuthMiddleware::class, VerifiedMiddleware::class]);

Route::prefix('admin')->middleware(AuthMiddleware::class)->group(function () {
    Route::get('/users', [AdminController::class, 'users']);
});
```

O middleware do grupo é executado antes do middleware da própria rota. O middleware de rota só é executado depois de uma rota corresponder, após o [model binding](routing.md#model-binding-nas-rotas).

### Middleware com opções

Para configurar um middleware rota a rota, passe uma instância em vez do nome da classe:

```php
Route::post('/login', [AuthController::class, 'login'])
    ->middleware(new RateLimitMiddleware($limiter, maxAttempts: 5, decaySeconds: 60));
```

As instâncias não podem ser guardadas na [cache de rotas](routing.md#cache-de-rotas). Quando isso for relevante, crie uma pequena subclasse com as opções no construtor e registe-a pelo nome da classe (consulte o exemplo de [rate limiting](#rate-limiting)).

## Middleware PSR-15

O middleware de rota também pode implementar `Psr\Http\Server\MiddlewareInterface`, passado como instância ou como nome de classe. O Luany converte o pedido para PSR-7 e de volta automaticamente:

```php
Route::get('/api/data', [ApiController::class, 'data'])
    ->middleware(new \Middlewares\Cors($analyzer));
```

A ponte precisa de uma implementação PSR-7: `nyholm/psr7` e `guzzlehttp/psr7` são detetados automaticamente. Consulte [Interoperabilidade PSR](psr.md).

A lista global do Kernel só aceita middleware do Luany. Envolva um middleware PSR-15 em `Luany\Core\Psr\Psr15Middleware` para o usar globalmente:

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

O `Luany\Core\Middleware\CorsMiddleware` responde aos pedidos `OPTIONS` de preflight com `204 No Content` e adiciona cabeçalhos CORS a todas as outras respostas.

Registe-o **globalmente**: um pedido de preflight não corresponde a nenhuma rota, por isso o middleware de rota nunca o veria.

```php
protected array $middleware = [
    \Luany\Core\Middleware\CorsMiddleware::class,
    CsrfMiddleware::class,
];
```

Sem argumentos, permite todas as origens, sem credenciais. Para o configurar, associe-o no método `register()` de um provider — o Kernel constrói o middleware global através do container:

```php
use Luany\Core\Middleware\CorsMiddleware;

$app->bind(CorsMiddleware::class, fn () => new CorsMiddleware(
    allowedOrigins:   ['https://app.example.com', 'https://*.example.com'],
    allowCredentials: true,
    exposedHeaders:   ['X-Total-Count'],
));
```

| Opção | Predefinição |
| --- | --- |
| `allowedOrigins` | `['*']` |
| `allowedMethods` | `['GET', 'POST', 'PUT', 'PATCH', 'DELETE', 'OPTIONS']` |
| `allowedHeaders` | `['Content-Type', 'Authorization', 'X-Requested-With']` |
| `exposedHeaders` | `[]` |
| `allowCredentials` | `false` |
| `maxAge` | `86400` (segundos durante os quais um preflight pode ficar em cache) |

As origens podem ser exatas (`https://app.example.com`) ou subdomínios com wildcard: `https://*.example.com` (só HTTPS) ou `*.example.com` (qualquer esquema). Uma origem correspondente é devolvida com `Vary: Origin`; uma origem não permitida não recebe cabeçalhos CORS.

> **Aviso** `allowCredentials: true` combinado com a origem `*` lança uma `InvalidArgumentException` — permitiria que qualquer site fizesse pedidos autenticados. Liste as origens explicitamente.

## Rate limiting

O `Luany\Core\Middleware\RateLimitMiddleware` limita os pedidos por IP de cliente numa janela temporal, usando um armazenamento `RateLimiterInterface`:

| Armazenamento | Utilização |
| --- | --- |
| `Luany\Core\RateLimit\FileRateLimiter` | Um ficheiro JSON por chave numa pasta; deploys num único servidor |
| `Luany\Core\RateLimit\InMemoryRateLimiter` | Dura apenas um processo PHP — testes e workers de longa duração |

```php
use Luany\Core\Middleware\RateLimitMiddleware;
use Luany\Core\RateLimit\FileRateLimiter;

$limiter = new FileRateLimiter(base_path('storage/rate-limits'));

Route::post('/login', [AuthController::class, 'login'])
    ->middleware(new RateLimitMiddleware($limiter, maxAttempts: 5, decaySeconds: 60, keyPrefix: 'login'));
```

| Argumento | Predefinição | Significado |
| --- | --- | --- |
| `limiter` | — | O armazenamento |
| `maxAttempts` | `60` | Pedidos permitidos por janela |
| `decaySeconds` | `60` | Duração da janela |
| `keyPrefix` | `'rl'` | Separa os contadores entre rotas |

As respostas permitidas recebem os cabeçalhos `X-RateLimit-Limit` e `X-RateLimit-Remaining`. Acima do limite, o middleware devolve `429 Too Many Requests` com `Retry-After`.

A chave do contador é `{prefix}:{ip}`. Sobreponha `keyFor()` para limitar por utilizador ou por token de API, e defina as opções no construtor para que o middleware possa ser registado pelo nome da classe:

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

## Proxies de confiança

Atrás de um load balancer ou de um reverse proxy, o `REMOTE_ADDR` é o endereço do proxy. Indique ao pedido em que proxies confiar, para que `$request->ip()` e `$request->isSecure()` leiam `X-Forwarded-For` e `X-Forwarded-Proto` — por exemplo, no `bootstrap/app.php` ou no método `boot()` de um provider:

```php
use Luany\Core\Http\Request;

Request::setTrustedProxies(['10.0.0.0/8', '127.0.0.1', '::1']);
```

São aceites IPs e intervalos CIDR (IPv4 e IPv6). Os cabeçalhos reencaminhados vindos de qualquer outro endereço são ignorados, por isso os clientes não conseguem falsificar o seu IP — o que é importante para o rate limiting.

> **Nota** A flag `secure` automática do cookie de sessão não usa estes cabeçalhos. Atrás de um proxy que termina o TLS, defina `'secure' => true` em [`config/session.php`](session.md#configuração).
