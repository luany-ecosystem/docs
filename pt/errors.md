# Tratamento de erros

Todas as exceções lançadas durante o tratamento de um pedido são apanhadas pelo Kernel e convertidas numa resposta. Os resultados HTTP esperados (404, 403, 422…) recebem o status correto; os erros inesperados são registados no log e apresentados como uma página 500 — ou como uma página de debug detalhada em desenvolvimento.

## Como as exceções se tornam respostas

| Exceção | Cliente JSON | Browser |
| --- | --- | --- |
| `ValidationException` | 422 `{"message", "errors"}` | Redirecionamento para trás, erros em flash — consulte [Validação](validation.md) |
| `HttpException` (`abort()`) | `{"message": "…"}` com o respetivo status e cabeçalhos | Apresentada pelo Handler |
| Nenhuma rota correspondente | 404 `{"message": "Not Found"}` | Apresentada pelo Handler |
| Método errado | 405 com cabeçalho `Allow` | Apresentada pelo Handler |
| Qualquer outra | Reportada e apresentada pelo Handler | Reportada e apresentada pelo Handler |

Um "cliente JSON" é um pedido cujo cabeçalho `Accept` contém `application/json`, ou um pedido AJAX (`X-Requested-With: XMLHttpRequest`).

## Interromper um pedido

O `abort()` interrompe o pedido com um erro HTTP a partir de qualquer ponto do seu código:

```php
abort(404);
abort(403, 'You cannot edit this post.');
abort(429, 'Slow down.', ['Retry-After' => '60']);
```

Lança uma `Luany\Framework\Exceptions\HttpException`; também a pode lançar diretamente:

```php
use Luany\Framework\Exceptions\HttpException;

throw new HttpException(409, 'This slug is already taken.');
```

Sem mensagem, é usada uma razão padrão: `Bad Request`, `Unauthorized`, `Forbidden`, `Not Found`, `Method Not Allowed`, `Unprocessable Content`, `Too Many Requests`, `Internal Server Error`, ou `HTTP Error` para outros códigos. As mensagens são escapadas em HTML quando apresentadas, por isso podem conter dados do utilizador com segurança.

## O handler de exceções

O `app/Exceptions/Handler.php` estende `Luany\Framework\Exceptions\Handler` e é associado no `bootstrap/app.php`, com o modo de debug obtido de `APP_DEBUG`:

```php
$app->singleton(
    \Luany\Framework\Exceptions\Handler::class,
    fn() => new App\Exceptions\Handler((bool) Env::get('APP_DEBUG', false))
);
```

Para cada exceção, o Kernel chama `report()` e depois `render()`.

### Apresentação

O método `render()` base produz:

| Status | Debug desligado | Debug ligado |
| --- | --- | --- |
| 4xx | `<h1>404 — Not Found</h1>` | Igual |
| 5xx | `<h1>500 — Server Error</h1>` | Página de debug: classe da exceção, mensagem, ficheiro, linha, pedido e stack trace |

O Handler do skeleton apresenta as suas próprias vistas:

| Status | Vista |
| --- | --- |
| 404 | `views/errors/404.lte` |
| 5xx | `views/errors/500.lte` (a página de debug quando `APP_DEBUG=true`) |
| Outros 4xx | Recorre ao handler base |

Edite essas vistas à vontade — estendem o layout principal. Para adicionar páginas para outros status, procure a vista pelo código de status:

```php
public function render(\Throwable $e): Response
{
    $status = $this->statusCodeFor($e);

    if ($status >= 500 && $this->debug) {
        return parent::render($e);
    }

    $view = is_file(app()->viewsPath("errors/{$status}.lte"))
        ? "errors.{$status}"
        : ($status >= 500 && is_file(app()->viewsPath('errors/500.lte')) ? 'errors.500' : null);

    if ($view === null) {
        return parent::render($e);
    }

    return Response::make(view($view, ['exception' => $e]), $status)
        ->withHeaders($this->headersFor($e));
}
```

Depois crie `views/errors/403.lte`, `views/errors/419.lte` e assim por diante.

| Método auxiliar | Devolve |
| --- | --- |
| `statusCodeFor($e)` | O status da `HttpException`, 422 para validação, 404 / 405 para routing, caso contrário 500 |
| `headersFor($e)` | `Allow` para 405, os cabeçalhos da `HttpException`, caso contrário `[]` |

> **Aviso** Nunca ative o `APP_DEBUG` em produção. A página de debug revela caminhos de ficheiros, a mensagem da exceção e o stack trace.

### Reporte de exceções

O `report()` regista a exceção através do [logger](logging.md) ao nível `error`, com a exceção (e o respetivo trace) no contexto. As exceções cuja classe consta de `$dontReport` são ignoradas. A lista predefinida do framework é:

```php
protected array $dontReport = [
    \Luany\Framework\Exceptions\HttpException::class,
    \Luany\Framework\Exceptions\ValidationException::class,
    \Luany\Core\Exceptions\RouteNotFoundException::class,
    \Luany\Core\Exceptions\MethodNotAllowedException::class,
];
```

> **Nota** O Handler do skeleton declara `protected array $dontReport = [];`, o que **substitui** essa lista — por isso os 404 e as chamadas a `abort()` ficam registados no log como erros. Copie a lista acima para o seu Handler para manter os erros HTTP esperados fora do log, e acrescente as suas próprias classes.

Sobreponha o `report()` para enviar as exceções para outro destino:

```php
public function report(\Throwable $e): void
{
    if (!$this->shouldReport($e)) {
        return;
    }

    parent::report($e);               // continua a escrever no log
    \Sentry\captureException($e);
}
```

As exceções lançadas dentro do `report()` são ignoradas, para que um reporter avariado nunca esconda a página de erro original.

## APIs JSON

Os erros HTTP e os erros de validação já são devolvidos em JSON aos clientes JSON. As outras exceções passam pelo `Handler::render()`, que não tem acesso ao pedido; se a sua API precisar de um 500 em JSON, verifique aí o cabeçalho `Accept`:

```php
public function render(\Throwable $e): Response
{
    if (str_contains($_SERVER['HTTP_ACCEPT'] ?? '', 'application/json')) {
        return Response::json(['message' => $this->debug ? $e->getMessage() : 'Server Error'], $this->statusCodeFor($e));
    }

    return parent::render($e);
}
```

## Quando o próprio handler falha

Se nenhum Handler estiver associado, ou se o `render()` lançar uma exceção, o Kernel recorre a respostas simples: o status e a mensagem escapada da exceção HTTP, `405` com `Allow`, `404`, ou um 500 genérico (`{"message": "Server Error"}` para clientes JSON), e regista no log a exceção inesperada.
