# Ciclo de vida do pedido

Esta página acompanha um pedido desde o `public/index.php` até à resposta enviada ao browser. Conhecer a ordem destes passos deixa claro onde registar serviços, middleware e rotas.

## Visão geral

```
public/index.php
  └─ bootstrap/app.php     autoload → Application → .env → providers register()
  └─ Kernel::boot()        serviços predefinidos → rotas → providers boot()
  └─ Kernel::handle()      middleware global → router → middleware de rota → ação
  └─ Response::send()
  └─ Kernel::terminate()
```

## 1. O front controller

Todos os pedidos entram pelo `public/index.php`:

```php
$app = require dirname(__DIR__) . '/bootstrap/app.php';

$kernel   = $app->make(Kernel::class);
$kernel->boot();

$request  = Request::fromGlobals();
$response = $kernel->handle($request);
$response->send();

$kernel->terminate($request, $response);
```

## 2. Bootstrapping

O `bootstrap/app.php` cria a aplicação e devolve-a:

1. Carrega o autoloader do Composer.
2. Cria `Luany\Framework\Application` — o [container de serviços](container.md). Este regista-se a si próprio, pelo que `app()` funciona a partir daqui.
3. Carrega o `.env` com `Env::load()` e garante a existência de `APP_ENV` e `APP_URL` com `Env::required()`.
4. Regista os service providers. `$app->register()` chama o método `register()` de cada provider **imediatamente**; o `boot()` é adiado.
5. Associa `App\Http\Kernel` como Kernel HTTP e `App\Exceptions\Handler` como handler de erros.

## 3. Arranque do Kernel

O `Kernel::boot()` é executado uma única vez, por esta ordem:

| Passo | O que acontece |
| --- | --- |
| 1. `config` | Carrega todos os ficheiros de `config/` |
| 2. `session` | Regista a sessão em ficheiro (iniciada de forma preguiçosa, na primeira utilização) |
| 3. `csrf` | Regista o serviço de token CSRF |
| 4. `view` | Regista o motor LTE, define a flag de debug do LTE, aplica `app.timezone` e liga o `Route::view()` |
| 5. `translator` | Regista o tradutor a partir de `app.locale` e das chaves relacionadas |
| 6. `log` | Regista o logger em ficheiro PSR-3 |
| 7. Resolver | Faz com que o router construa controllers e middleware através do container e chame as ações com [injeção nos métodos](controllers.md#injeção-nos-métodos) |
| 8. Rotas | Faz require do `routes/http.php` e depois de todos os outros `routes/*.php`, por ordem alfabética |
| 9. Providers | Chama `boot()` em todos os providers registados |
| 10. Validação | Liga as regras `unique` / `exists` ao binding `db`, se existir |

### Serviços predefinidos

Os passos 1–6 só registam um serviço quando nenhum provider o fez antes. Cada serviço está disponível sob uma chave e um nome de classe:

| Chave | Classe / interface | Predefinição |
| --- | --- | --- |
| `config` | `Luany\Framework\Support\Config` | Ficheiros em `config/` |
| `session` | `Luany\Framework\Contracts\SessionInterface` | `FileSession` em `storage/sessions` |
| `csrf` | `Luany\Framework\Security\CsrfToken` | Token guardado na sessão |
| `view` | `Luany\Lte\Engine` | `views/`, cache em `storage/cache/views` |
| `translator` | `Luany\Framework\Support\Translator` | `lang/{locale}.php` |
| `log` | `Psr\Log\LoggerInterface` | `FileLogger` em `storage/logs` |

Para substituir um deles, associe a chave ou a classe no método `register()` de um provider; o Kernel cria um alias do outro nome para o seu binding:

```php
public function register(Application $app): void
{
    $app->singleton('log', fn () => new \Monolog\Logger('app'));
}
```

Como as rotas são carregadas **antes** do arranque dos providers, o método `boot()` de um provider pode inspecionar ou adicionar rotas, e todos os serviços de que um ficheiro de rotas precisa no momento do carregamento têm de ser associados no `register()`.

## 4. Tratamento do pedido

`Request::fromGlobals()` constrói o [pedido](requests.md) a partir das variáveis globais do PHP: aplica o method spoofing através de `_method`, remove o caminho base da subpasta e descodifica corpos JSON.

Em seguida, o `Kernel::handle()`:

1. Passa o pedido pelo **middleware global** listado em `App\Http\Kernel::$middleware` (no skeleton: live reload e depois CSRF). O middleware global é executado em todos os pedidos, incluindo os que não correspondem a nenhuma rota.
2. Entrega-o ao **router**, que procura a primeira rota cujo URI e método correspondem. Um URI correspondente com o método errado lança um 405; sem correspondência, lança um 404.
3. Resolve os parâmetros da rota (incluindo o [model binding](routing.md#model-binding-nas-rotas)).
4. Executa o **middleware de rota** (primeiro o do grupo, depois o da própria rota).
5. Chama a **ação** — uma closure ou um método de controller — resolvendo os argumentos através do container.
6. Converte o valor devolvido numa `Response`: strings tornam-se HTML, arrays e objetos tornam-se JSON.

## 5. Exceções

Qualquer exceção lançada durante os passos 1–6 é convertida pelo Kernel numa resposta:

| Exceção | Resposta |
| --- | --- |
| `ValidationException` | 422 em JSON para clientes JSON; caso contrário, redirecionamento para trás com os erros em flash |
| Exceções HTTP (`abort()`, 404, 405) para clientes JSON | JSON `{"message": "…"}` com o status correto |
| Qualquer outra | Reportada e apresentada pelo `App\Exceptions\Handler` |

Consulte [Tratamento de erros](errors.md).

## 6. Envio e terminação

`Response::send()` escreve a linha de status, os cabeçalhos, os cookies e o corpo. Por fim, é chamado o `Kernel::terminate()` — um hook vazio que pode sobrepor em `App\Http\Kernel` para fazer trabalho depois de a resposta ser enviada:

```php
public function terminate(Request $request, Response $response): void
{
    logger('Handled {method} {uri}', ['method' => $request->method(), 'uri' => $request->uri()]);
}
```

## Tratar pedidos em testes

Os mesmos passos funcionam sem servidor web — construa um `Request` manualmente e passe-o ao Kernel:

```php
$app    = require dirname(__DIR__, 2) . '/bootstrap/app.php';
$kernel = $app->make(\Luany\Framework\Http\Kernel::class);
$kernel->boot();

$response = $kernel->handle(new Request('GET', '/', server: ['REMOTE_ADDR' => '127.0.0.1']));

$this->assertSame(200, $response->getStatusCode());
```
