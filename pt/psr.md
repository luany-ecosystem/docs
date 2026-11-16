# Interoperabilidade PSR

O Luany integra-se no ecossistema PHP mais alargado através das normas do PHP-FIG: mensagens PSR-7, middleware e request handlers PSR-15, fábricas PSR-17 e containers PSR-11. Esta página mostra como converter mensagens, reutilizar middleware PSR-15, executar a aplicação atrás de um servidor PSR-15 e ligar outro container.

## Requisitos

O `luany/core` depende apenas das **interfaces** PSR. As pontes precisam de uma implementação PSR-7 / PSR-17; instale uma:

```bash
composer require nyholm/psr7
```

O `nyholm/psr7` e o `guzzlehttp/psr7` são detetados automaticamente, por esta ordem. Qualquer outra implementação (por exemplo, `laminas/diactoros`) pode ser configurada explicitamente — consulte [Fábricas PSR-17](#fábricas-psr-17).

Sem uma implementação, a primeira conversão lança uma `LogicException` que explica o que instalar.

## Conversão PSR-7

`Request` e `Response` convertem-se de e para mensagens PSR-7:

```php
use Luany\Core\Http\Request;
use Luany\Core\Http\Response;

$psrRequest  = $request->toPsr();             // Psr\Http\Message\ServerRequestInterface
$request     = Request::fromPsr($psrRequest);

$psrResponse = $response->toPsr();            // Psr\Http\Message\ResponseInterface
$response    = Response::fromPsr($psrResponse);
```

Use-a quando chamar uma biblioteca PSR-7 a partir de um controller:

```php
public function upload(Request $request): Response
{
    $psrResponse = $this->storageClient->forward($request->toPsr());

    return Response::fromPsr($psrResponse);
}
```

O que é transportado:

| Parte | Request | Response |
|---|---|---|
| Método, URI, query string | Sim (URI absoluto construído a partir de `Host` e `isSecure()`) | — |
| Cabeçalhos | Sim | Sim |
| Corpo interpretado (parsed body) | Sim; os corpos JSON também são escritos no stream | — |
| Stream do corpo | Os pedidos JSON são descodificados quando o corpo interpretado está vazio | Sim |
| Cookies | Cookie params | Cabeçalhos `Set-Cookie`, um por cookie |
| Server params | Sim | — |
| Ficheiros carregados | Campos de ficheiro único | — |
| Estado e versão do protocolo | — | Sim |

> **Nota** Os campos de ficheiros múltiplos (`name="photos[]"`) não são convertidos em ficheiros carregados PSR-7.

## Middleware PSR-15 nas rotas

Qualquer `Psr\Http\Server\MiddlewareInterface` pode ser associado a uma rota ou a um grupo, como instância ou como nome de classe. O pipeline envolve-o automaticamente em `Luany\Core\Psr\Psr15Middleware`:

```php
use Luany\Core\Routing\Route;

Route::get('/api/data', [ApiController::class, 'index'])
    ->middleware(new \Middlewares\Cors($analyzer));

Route::middleware(\Middlewares\ResponseTime::class)->group(function () {
    Route::get('/reports', [ReportController::class, 'index']);
});
```

Os nomes de classe são construídos através do container, pelo que as dependências do construtor são injetadas. As alterações que o middleware PSR-15 faz aos cabeçalhos, à query, ao corpo interpretado, aos cookies e aos server params do pedido chegam ao middleware seguinte e ao controller; a sua resposta regressa como uma `Response` do Luany.

> **Nota** Os *atributos* do pedido PSR-7 (`withAttribute()`) não passam para a `Request` do Luany, pelo que um middleware que comunique através de atributos (por exemplo, um middleware de autenticação que guarda um utilizador) não consegue passar esse valor ao seu controller.

## Middleware PSR-15 no Kernel

A lista global `$middleware` do Kernel HTTP também aceita middleware PSR-15, como nome de classe ou como instância, e envolve-o da mesma forma:

```php
protected array $middleware = [
    CsrfMiddleware::class,
    \Middlewares\ResponseTime::class,
];
```

É executado em todos os pedidos, antes do routing. Consulte [Middleware](middleware.md#middleware-psr-15).

## Executar atrás de um servidor PSR-15

O `Luany\Core\Psr\RequestHandler` expõe uma aplicação Luany como um `RequestHandlerInterface` PSR-15. Aceita o Kernel HTTP (qualquer objeto com `handle(Request): Response`), o router ou um callable:

```php
use Luany\Core\Psr\RequestHandler;

$handler     = new RequestHandler($kernel);
$psrResponse = $handler->handle($psrServerRequest);
```

### Worker do RoadRunner

O RoadRunner entrega cada pedido a um worker PHP de longa duração em formato PSR-7. Arranque a aplicação uma vez e depois sirva os pedidos num ciclo:

```php
<?php
// worker.php

use Luany\Core\Psr\RequestHandler;
use Luany\Framework\Http\Kernel;
use Nyholm\Psr7\Factory\Psr17Factory;
use Nyholm\Psr7\Response;
use Spiral\RoadRunner\Http\PSR7Worker;
use Spiral\RoadRunner\Worker;

$app = require __DIR__ . '/bootstrap/app.php';

$kernel = $app->make(Kernel::class);
$kernel->boot();

$handler = new RequestHandler($kernel);

$factory = new Psr17Factory();
$worker  = new PSR7Worker(Worker::create(), $factory, $factory, $factory);

while ($request = $worker->waitRequest()) {
    try {
        $worker->respond($handler->handle($request));
    } catch (\Throwable $e) {
        $worker->respond(new Response(500));
        $worker->getWorker()->error((string) $e);
    }
}
```

Requer `spiral/roadrunner-http` e `nyholm/psr7`.

### Modo worker do FrankenPHP

O modo worker do FrankenPHP preenche as superglobais em cada pedido, pelo que não é necessária nenhuma conversão PSR-7 — crie o pedido a partir das globais dentro do handler:

```php
<?php
// public/worker.php

use Luany\Core\Http\Request;
use Luany\Framework\Http\Kernel;

$app = require dirname(__DIR__) . '/bootstrap/app.php';

$kernel = $app->make(Kernel::class);
$kernel->boot();

$handle = static function () use ($kernel): void {
    $request  = Request::fromGlobals();
    $response = $kernel->handle($request);
    $response->send();
    $kernel->terminate($request, $response);
};

while (frankenphp_handle_request($handle)) {
    gc_collect_cycles();
}
```

> **Aviso** Num worker de longa duração, a aplicação, os seus singletons e o estado estático (rotas, definições do validador) vivem durante muitos pedidos. A sessão em ficheiros do framework e helpers como `validate()` leem a sessão nativa do PHP e `$_SERVER`, que um worker PSR-7 como o RoadRunner não preenche. As APIs JSON sem estado são a escolha segura para workers PSR-7; teste cuidadosamente as páginas que dependem da sessão antes de passar para produção.

Os uploads recebidos através de um worker PSR-7 não são uploads do PHP, pelo que as regras `file`, `image` e `mimes` os rejeitam até registar um verificador com `Validator::setUploadedFileVerifier()` — consulte [Validação](validation.md#testes-e-servidores-psr-7).

## Containers PSR-11

A `Application` do framework implementa `Psr\Container\ContainerInterface`, e o Kernel HTTP já a usa para construir controllers e middleware — não há nada a configurar. Consulte [Container de serviços](container.md#psr-11).

Quando usa o `luany/core` isoladamente com outro container PSR-11 (PHP-DI, Symfony DI, League Container…), deixe que o router e o pipeline construam as classes através dele:

```php
use Luany\Core\Middleware\Pipeline;

Pipeline::useContainer($container);
```

As classes que o container reporta com `has()` vêm de `get()`; qualquer outra classe é criada com `new`. Para ter controlo total, defina antes um callable de resolução:

```php
use Luany\Core\Middleware\Pipeline;
use Luany\Core\Routing\Route;

Pipeline::setDefaultResolver(fn (string $class): object => $container->get($class));
Route::router()->setResolver(fn (string $class): object => $container->get($class));
```

`Pipeline::setDefaultResolver()` aplica-se ao middleware e, através do router, aos controllers; `Router::setResolver()` sobrepõe-se a ele para um router concreto.

## Fábricas PSR-17

As pontes obtêm as suas fábricas de `Luany\Core\Psr\Psr17`. Configure uma implementação explicitamente com um objeto que implemente as interfaces de fábrica de server request, response e stream (e, opcionalmente, a de uploaded file), como o `Psr17Factory` do Nyholm:

```php
use Luany\Core\Psr\Psr17;

Psr17::use(new \Nyholm\Psr7\Factory\Psr17Factory());
```

Um objeto que não implemente as três interfaces obrigatórias é rejeitado com uma `InvalidArgumentException`. `Psr17::reset()` esquece as fábricas configuradas e volta a ativar a deteção automática.
