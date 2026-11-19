# Helpers

O luany/framework define um conjunto de funções globais para as tarefas do dia a dia. Cada uma só é declarada se ainda não existir uma função com o mesmo nome, pelo que uma aplicação pode definir primeiro a sua própria versão.

## Visão geral

| Área | Helpers |
| --- | --- |
| Aplicação | [`app`](#app), [`base_path`](#base_path), [`config`](#config), [`env`](#env) |
| HTTP | [`abort`](#abort), [`redirect`](#redirect), [`response`](#response) |
| URLs | [`url`](#url), [`route`](#route), [`asset`](#asset) |
| Vistas | [`view`](#view), [`e`](#e) |
| Sessão | [`session`](#session), [`flash`](#flash), [`get_flash`](#get_flash), [`old`](#old) |
| Segurança | [`csrf_token`](#csrf_token), [`csrf_field`](#csrf_field) |
| Validação | [`validate`](#validate) |
| Localização | [`__`](#__), [`locale`](#locale) |
| Logging | [`logger`](#logger) |

## Aplicação

### `app`

`app(?string $abstract = null): mixed` — o container da aplicação, ou um serviço resolvido a partir dele.

```php
app();                        // Luany\Framework\Application
app('session');               // um binding pela chave
app(ReportService::class);    // uma classe, com autowiring
```

Consulte [Container de serviços](container.md).

### `base_path`

`base_path(string $path = ''): string` — caminho absoluto dentro do projeto.

```php
base_path();                      // /var/www/my-app
base_path('storage/exports');     // /var/www/my-app/storage/exports
```

### `config`

`config(string|array|null $key = null, mixed $default = null): mixed`

```php
config('app.name');                   // leitura
config('app.missing', 'fallback');    // com valor predefinido
config(['app.debug' => false]);       // escrita (devolve null)
config();                             // o repositório Config
```

Consulte [Configuração](configuration.md#aceder-à-configuração).

### `env`

`env(string $key, mixed $default = null): mixed` — uma variável de ambiente. `true`, `false`, `null` e `empty` são convertidos. Consulte [Configuração](configuration.md#ler-valores).

```php
env('APP_ENV', 'production');
```

## HTTP

### `abort`

`abort(int $code, string $message = '', array $headers = []): never` — interrompe o pedido com um erro HTTP. Consulte [Tratamento de erros](errors.md#interromper-um-pedido).

```php
abort(404);
abort(403, 'Forbidden');
abort(429, 'Slow down', ['Retry-After' => '60']);
```

### `redirect`

`redirect(string $url, int $status = 302): Response`

```php
return redirect('/dashboard');
return redirect(route('posts.show', ['id' => $post->id]));
return redirect('/new-url', 301);
```

### `response`

`response(string $body = '', int $status = 200): Response`

```php
return response('<h1>Created</h1>', 201);
return response()->status(204);
```

Para JSON, use `Response::json()` ou devolva um array. Consulte [Pedidos e respostas](requests.md#criar-respostas).

## URLs

Os URLs são construídos a partir de `config('app.url')`, recorrendo a `APP_URL` em alternativa.

### `url`

`url(string $path = ''): string`

```php
url();             // http://localhost:8000
url('/posts/1');   // http://localhost:8000/posts/1
```

### `route`

`route(string $name, array $params = []): string` — URL de uma [rota com nome](routing.md#rotas-com-nome). Os parâmetros que não constam do padrão passam para a query string; um nome desconhecido lança uma `RuntimeException`.

```php
route('posts.show', ['id' => 5]);           // http://localhost:8000/posts/5
route('posts.index', ['page' => 2]);        // http://localhost:8000/posts?page=2
```

### `asset`

`asset(string $path): string` — caminho de um ficheiro em `public/assets`, relativo à raiz do site: `asset('css/app.css')` dá `/assets/css/app.css` (ou `/app/assets/css/app.css` quando o `APP_URL` tem uma subpasta). As páginas mantêm o CSS e o JavaScript em qualquer host ou porta — um segundo `luany serve`, um endereço da rede local, um proxy.

Defina `ASSET_URL` (ou `app.asset_url`) para uma base absoluta, por exemplo uma CDN ou links em emails: `ASSET_URL=https://cdn.example.com` dá `https://cdn.example.com/assets/css/app.css`. Requer luany/framework 1.5; as versões anteriores prefixam o `APP_URL`.

```lte
<link rel="stylesheet" href="{{ asset('css/app.css') }}">
```

## Vistas

### `view`

`view(string $name, array $data = []): string` — gera um template LTE como string.

```php
return view('pages.posts.index', ['posts' => $posts]);
```

### `e`

`e(mixed $value): string` — escapa um valor em HTML, tal como o `{{ }}` faz no LTE.

```php
echo '<p>' . e($comment->body) . '</p>';
```

## Sessão

### `session`

`session(?string $key = null, mixed $default = null): mixed` — a sessão, ou um valor dela. Para escrever, use o objeto de sessão.

```php
session('user_id');
session('theme', 'light');
session()->set('theme', 'dark');
```

Consulte [Sessão e mensagens flash](session.md).

### `flash`

`flash(string $type, string $message): void` — uma mensagem para o pedido seguinte.

```php
flash('success', 'Post saved.');
```

### `get_flash`

`get_flash(): ?array` — a mensagem guardada em flash no pedido anterior, no formato `['type' => ..., 'message' => ...]`, ou `null`.

### `old`

`old(string $key, mixed $default = null): mixed` — dados guardados em flash por uma validação falhada.

```lte
<input name="title" value="{{ old('title', $post->title) }}">
```

## Segurança

### `csrf_token`

`csrf_token(): string` — o token CSRF da sessão.

```lte
<meta name="csrf-token" content="{{ csrf_token() }}">
```

### `csrf_field`

`csrf_field(): string` — um input `_token` oculto. No LTE, o `@csrf` faz o mesmo.

```php
<form method="POST"><?= csrf_field() ?> ...</form>
```

Consulte [Proteção CSRF](csrf.md).

## Validação

### `validate`

`validate(array $data, array $rules, string $redirectTo = ''): array` — devolve os campos validados ou lança uma `ValidationException` que redireciona para trás (ou responde com 422 em JSON).

```php
$data = validate($request->body(), [
    'title' => 'required|string|max:255',
]);
```

Consulte [Validação](validation.md).

## Localização

### `__`

`__(string $key, array $replace = []): string` — traduz uma chave.

```php
__('welcome.title', ['name' => 'Ana']);
```

### `locale`

`locale(): string` — o código do idioma ativo.

Consulte [Localização](localization.md).

## Logging

### `logger`

`logger(string|\Stringable|null $message = null, array $context = [], string $level = 'debug'): ?LoggerInterface` — escreve um registo, ou devolve o logger quando chamado sem mensagem.

```php
logger('Import started');
logger('Low stock for {sku}', ['sku' => 'A-1'], 'warning');
logger()->error('Payment failed', ['order' => 7]);
```

Consulte [Logging](logging.md).
