# Pedidos e respostas

Todas as ações recebem um `Luany\Core\Http\Request` e produzem uma `Luany\Core\Http\Response`. Esta página descreve as duas APIs.

## Aceder ao pedido

Declare o tipo `Request` num método de controller ou numa closure:

```php
use Luany\Core\Http\Request;

public function store(Request $request): Response
{
    $title = $request->input('title');
    // ...
}
```

O Kernel cria-o com `Request::fromGlobals()`, que:

- aplica o **method spoofing**: um `POST` com um campo `_method` igual a `PUT`, `PATCH` ou `DELETE` passa a usar esse método;
- remove do caminho a pasta do front controller, para que as aplicações numa subpasta vejam `/posts` e não `/myapp/posts`;
- descodifica **corpos JSON** quando o `Content-Type` contém `application/json`.

## Método e URI

| Método | Devolve |
| --- | --- |
| `method()` | Método em maiúsculas, após o spoofing: `'GET'`, `'PUT'`, … |
| `isMethod('post')` | Comparação sem distinção entre maiúsculas e minúsculas |
| `isGet()`, `isPost()`, `isPut()`, `isPatch()`, `isDelete()` | `bool` |
| `uri()` | Caminho sem query string, p. ex. `/posts/5` |
| `url()` | URL absoluto sem query string: esquema, `Host` e caminho |
| `isAjax()` | `X-Requested-With: XMLHttpRequest` |
| `expectsJson()` | `Accept` contém `application/json`, ou é um pedido AJAX |

## Dados de entrada

| Método | Descrição |
| --- | --- |
| `input($key, $default = null)` | Valor do corpo, depois da query string, depois `$default` |
| `query($key, $default = null)` | Apenas a query string |
| `post($key, $default = null)` | Apenas o corpo (campos do formulário ou JSON descodificado) |
| `body()` | O corpo inteiro como array |
| `queryParams()` | A query string inteira como array |
| `all()` | Query string e corpo combinados (o corpo prevalece) |
| `only(['a', 'b'])` / `except(['c'])` | Subconjunto de `all()` |
| `has($key)` | Chave presente e diferente de `null`, no corpo ou na query string |
| `filled($key)` | Presente e diferente de `''` |

```php
$page   = (int) $request->query('page', 1);
$fields = $request->only(['title', 'body']);

if ($request->filled('search')) {
    // ...
}
```

> **Dica** Valide `$request->body()` em vez de `all()`, para que não seja possível infiltrar valores da query string nos seus dados: `validate($request->body(), $rules)`.

## Ficheiros

| Método | Descrição |
| --- | --- |
| `file($key)` | A entrada de `$_FILES` (array) ou `null` |
| `hasFile($key)` | Existe um upload e não tem erros |
| `files()` | Todos os ficheiros enviados |

```php
if ($request->hasFile('avatar')) {
    $file = $request->file('avatar');
    move_uploaded_file($file['tmp_name'], base_path('storage/avatars/' . bin2hex(random_bytes(8))));
}
```

## Cabeçalhos, cookies e valores do servidor

| Método | Descrição |
| --- | --- |
| `header($name, $default = null)` | Sem distinção entre maiúsculas e minúsculas: `header('content-type')` |
| `headers()` | Todos os cabeçalhos |
| `cookie($name, $default = null)` | O valor de um cookie |
| `hasCookie($name)` / `cookies()` | Verificação de existência / todos os cookies |
| `server($key, $default = null)` / `serverParams()` | Valores de `$_SERVER` |
| `ip()` | IP do cliente — só respeita `X-Forwarded-For` vindo de [proxies de confiança](middleware.md#proxies-de-confiança) |
| `isSecure()` | HTTPS — só respeita `X-Forwarded-Proto` vindo de proxies de confiança |
| `userAgent()` | Cabeçalho `User-Agent` |

## Criar respostas

Devolva uma string ou um array nos casos simples; construa uma `Response` para ter controlo total.

```php
use Luany\Core\Http\Response;

return Response::make('<h1>Hello</h1>');
return Response::make('Created', 201, ['X-Id' => '42']);
return Response::json(['id' => 42], 201);
return Response::redirect('/login');
return Response::redirect('/new-home', 301);
```

| Factory | Predefinição |
| --- | --- |
| `make($body = '', $status = 200, $headers = [])` | Resposta simples |
| `json($data, $status = 200, $headers = [], $flags)` | `Content-Type: application/json; charset=UTF-8`; barras e Unicode sem escape |
| `redirect($url, $status = 302)` | Cabeçalho `Location` |
| `notFound()`, `unauthorized()`, `forbidden()`, `serverError()` | 404 / 401 / 403 / 500 com um corpo HTML mínimo |

O `json()` substitui UTF-8 inválido em vez de devolver um corpo vazio; qualquer outro erro de codificação lança `JsonException`.

Os [helpers](helpers.md) `response()` e `redirect()` são atalhos para `make()` e `redirect()`.

### Métodos fluentes

```php
return Response::make(view('pages.report'))
    ->status(200)
    ->header('Cache-Control', 'no-store')
    ->withHeaders(['X-Frame-Options' => 'DENY']);
```

| Método | Descrição |
| --- | --- |
| `status($code)` | Define o código de status |
| `body($string)` | Substitui o corpo |
| `header($name, $value)` / `withHeaders([...])` | Define cabeçalhos |
| `setProtocolVersion('2')` | Versão HTTP na linha de status |
| `getStatusCode()`, `getBody()`, `getHeaders()`, `getCookies()` | Leitura |
| `isRedirect()`, `isSuccessful()` | Verificações de status |

Estes métodos modificam o mesmo objeto de resposta e devolvem-no.

## Cookies

```php
return redirect('/')
    ->cookie('theme', 'dark', time() + 60 * 60 * 24 * 365, secure: $request->isSecure());
```

```php
cookie(
    string $name,
    string $value,
    int $expires = 0,          // timestamp Unix; 0 = cookie de sessão
    string $path = '/',
    string $domain = '',
    bool $secure = false,
    bool $httpOnly = true,
    string $sameSite = 'Lax'   // Lax | Strict | None
)
```

Por predefinição, os cookies são `HttpOnly` e `SameSite=Lax`. `SameSite=None` obriga a `Secure`. Nomes ou valores de SameSite inválidos lançam `InvalidArgumentException`. Chame `cookie()` várias vezes para definir vários cookies.

| Método | Descrição |
| --- | --- |
| `forgetCookie($name, $path = '/', $domain = '')` | Faz expirar um cookie |
| `rawCookie($setCookie)` | Acrescenta um valor `Set-Cookie` já formatado |

Leia os cookies com `$request->cookie('theme')`.

## PSR-7

Ambas as classes convertem de e para mensagens PSR-7:

```php
$psrRequest  = $request->toPsr();
$request     = Request::fromPsr($psrRequest);

$psrResponse = $response->toPsr();
$response    = Response::fromPsr($psrResponse);
```

Isto precisa de uma factory PSR-17: `nyholm/psr7` ou `guzzlehttp/psr7` são detetados automaticamente; em alternativa, chame `Luany\Core\Psr\Psr17::use($factory)`. Consulte [Interoperabilidade PSR](psr.md).
