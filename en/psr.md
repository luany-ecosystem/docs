# PSR Interoperability

Luany works with the wider PHP ecosystem through the PHP-FIG standards: PSR-7 messages, PSR-15 middleware and request handlers, PSR-17 factories and PSR-11 containers. This page shows how to convert messages, reuse PSR-15 middleware, run the application behind a PSR-15 server and plug in another container.

## Requirements

`luany/core` depends only on the PSR **interfaces**. The bridges need a PSR-7 / PSR-17 implementation; install one:

```bash
composer require nyholm/psr7
```

`nyholm/psr7` and `guzzlehttp/psr7` are detected automatically, in that order. Any other implementation (for example `laminas/diactoros`) can be configured explicitly — see [PSR-17 factories](#psr-17-factories).

Without an implementation, the first conversion throws a `LogicException` explaining what to install.

## PSR-7 conversion

`Request` and `Response` convert to and from PSR-7 messages:

```php
use Luany\Core\Http\Request;
use Luany\Core\Http\Response;

$psrRequest  = $request->toPsr();             // Psr\Http\Message\ServerRequestInterface
$request     = Request::fromPsr($psrRequest);

$psrResponse = $response->toPsr();            // Psr\Http\Message\ResponseInterface
$response    = Response::fromPsr($psrResponse);
```

Use this when calling a PSR-7 library from a controller:

```php
public function upload(Request $request): Response
{
    $psrResponse = $this->storageClient->forward($request->toPsr());

    return Response::fromPsr($psrResponse);
}
```

What is carried across:

| Part | Request | Response |
|---|---|---|
| Method, URI, query string | Yes (absolute URI built from `Host` and `isSecure()`) | — |
| Headers | Yes | Yes |
| Parsed body | Yes; JSON bodies are also written to the stream | — |
| Body stream | JSON requests are decoded when the parsed body is empty | Yes |
| Cookies | Cookie params | `Set-Cookie` headers, one per cookie |
| Server params | Yes | — |
| Uploaded files | Single-file inputs | — |
| Status and protocol version | — | Yes |

> **Note** Multi-file inputs (`name="photos[]"`) are not converted to PSR-7 uploaded files.

## PSR-15 middleware in routes

Any `Psr\Http\Server\MiddlewareInterface` can be attached to a route or group, as an instance or a class name. The pipeline wraps it in `Luany\Core\Psr\Psr15Middleware` automatically:

```php
use Luany\Core\Routing\Route;

Route::get('/api/data', [ApiController::class, 'index'])
    ->middleware(new \Middlewares\Cors($analyzer));

Route::middleware(\Middlewares\ResponseTime::class)->group(function () {
    Route::get('/reports', [ReportController::class, 'index']);
});
```

Class names are built through the container, so their constructor dependencies are injected. Changes the PSR-15 middleware makes to the request's headers, query, parsed body, cookies and server params reach the next middleware and the controller; its response comes back as a Luany `Response`.

> **Note** PSR-7 request *attributes* (`withAttribute()`) are not carried over to the Luany `Request`, so middleware that communicates through attributes (for example authentication middleware storing a user) cannot pass that value to your controller.

## PSR-15 middleware in the Kernel

The HTTP Kernel's global `$middleware` list accepts PSR-15 middleware too, as a class name or an instance, and wraps it the same way:

```php
protected array $middleware = [
    CsrfMiddleware::class,
    \Middlewares\ResponseTime::class,
];
```

It runs on every request, before routing. See [Middleware](middleware.md#psr-15-middleware).

## Running behind a PSR-15 server

`Luany\Core\Psr\RequestHandler` exposes a Luany application as a PSR-15 `RequestHandlerInterface`. It accepts the HTTP Kernel (any object with `handle(Request): Response`), the router, or a callable:

```php
use Luany\Core\Psr\RequestHandler;

$handler     = new RequestHandler($kernel);
$psrResponse = $handler->handle($psrServerRequest);
```

### RoadRunner worker

RoadRunner hands each request to a long-running PHP worker as PSR-7. Boot the application once, then serve requests in a loop:

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

Requires `spiral/roadrunner-http` and `nyholm/psr7`.

### FrankenPHP worker mode

FrankenPHP's worker mode populates the superglobals for each request, so no PSR-7 conversion is needed — create the request from globals inside the handler:

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

> **Warning** In a long-running worker, the application, its singletons and static state (routes, validator settings) live for many requests. The framework's file session and helpers such as `validate()` read PHP's native session and `$_SERVER`, which a PSR-7 worker like RoadRunner does not populate. Stateless JSON APIs are the safe fit for PSR-7 workers; test session-based pages carefully before going to production.

Uploads received through a PSR-7 worker are not PHP uploads, so the `file`, `image` and `mimes` rules reject them until you register a verifier with `Validator::setUploadedFileVerifier()` — see [Validation](validation.md#testing-and-psr-7-servers).

## PSR-11 containers

The framework's `Application` implements `Psr\Container\ContainerInterface`, and the HTTP Kernel already uses it to build controllers and middleware — nothing to configure. See [Service Container](container.md#psr-11).

When you use `luany/core` on its own with another PSR-11 container (PHP-DI, Symfony DI, League Container…), let the router and pipeline build classes through it:

```php
use Luany\Core\Middleware\Pipeline;

Pipeline::useContainer($container);
```

Classes the container reports with `has()` come from `get()`; any other class is created with `new`. For full control, set a resolver callable instead:

```php
use Luany\Core\Middleware\Pipeline;
use Luany\Core\Routing\Route;

Pipeline::setDefaultResolver(fn (string $class): object => $container->get($class));
Route::router()->setResolver(fn (string $class): object => $container->get($class));
```

`Pipeline::setDefaultResolver()` applies to middleware and, through the router, to controllers; `Router::setResolver()` overrides it for one router.

## PSR-17 factories

The bridges get their factories from `Luany\Core\Psr\Psr17`. Configure an implementation explicitly with one object that implements the server-request, response and stream factory interfaces (and optionally the uploaded-file factory), such as Nyholm's `Psr17Factory`:

```php
use Luany\Core\Psr\Psr17;

Psr17::use(new \Nyholm\Psr7\Factory\Psr17Factory());
```

An object that does not implement the three required interfaces is rejected with an `InvalidArgumentException`. `Psr17::reset()` forgets the configured factories and re-enables auto-detection.
