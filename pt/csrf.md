# Proteção CSRF

O cross-site request forgery leva o browser de um utilizador autenticado a submeter um pedido à sua aplicação sem o seu conhecimento. O Luany impede-o exigindo um token secreto, por sessão, em todos os pedidos que alteram estado.

## Como funciona

O skeleton regista o `App\Http\Middleware\CsrfMiddleware` como middleware global. Em cada pedido `POST`, `PUT`, `PATCH` e `DELETE`, compara o token submetido com o que está guardado na sessão. Os pedidos `GET`, `HEAD` e `OPTIONS` nunca são verificados.

| | Nome |
| --- | --- |
| Campo do formulário | `_token` (o campo antigo `csrf_token` também é aceite) |
| Cabeçalho HTTP | `X-CSRF-TOKEN` |
| Chave de sessão | `_csrf_token` |

O token é lido a partir da primeira string não vazia por esta ordem — campo do formulário, campo antigo, cabeçalho — e comparado em tempo constante.

O token mantém-se igual durante toda a sessão, por isso vários separadores, o botão de retroceder e pedidos AJAX simultâneos continuam a funcionar. Não é renovado a cada pedido.

## Formulários

Adicione o token a todos os formulários que não usem `GET`:

```lte
<form method="POST" action="/posts">
    @csrf
    <input name="title">
    <button>Save</button>
</form>
```

O `@csrf` gera um input `_token` oculto. Fora do LTE, use os helpers:

```php
<?= csrf_field() ?>        // <input type="hidden" name="_token" value="...">
<?= csrf_token() ?>        // o próprio token
```

## Pedidos em JavaScript

O layout do skeleton publica o token numa meta tag:

```lte
<meta name="csrf-token" content="{{ csrf_token() }}">
```

Envie-o no cabeçalho `X-CSRF-TOKEN`:

```js
const token = document.querySelector('meta[name="csrf-token"]').content;

await fetch('/posts', {
    method: 'POST',
    headers: {
        'Content-Type': 'application/json',
        'Accept': 'application/json',
        'X-CSRF-TOKEN': token,
    },
    body: JSON.stringify({ title: 'Hello' }),
});
```

## Falha na verificação

| Pedido | Resposta |
| --- | --- |
| Espera JSON | `{"message": "CSRF token mismatch."}` |
| Caso contrário | Texto simples `CSRF token mismatch.` |

O status é o `$failureStatus` do middleware: **419 Page Expired** no skeleton (a classe base do framework usa 403 por predefinição). Um token em falta, vazio ou que não seja uma string (por exemplo `_token[]=x`) é tratado como não correspondente, nunca como um erro de servidor.

Uma falha de correspondência significa normalmente que a sessão expirou ou que o formulário foi gerado antes de a sessão ser regenerada — recarregue a página para obter um token novo.

## Excluir URIs

Os webhooks de terceiros não podem conhecer o seu token. Liste os respetivos caminhos em `$except`:

```php
class CsrfMiddleware extends BaseCsrfMiddleware
{
    protected array $except = [
        '/stripe/webhook',     // caminho exato
        '/webhooks/*',         // qualquer caminho que comece por /webhooks/
    ];

    protected int $failureStatus = 419;
}
```

Um padrão corresponde ao caminho exato ou — com um `*` no fim — a qualquer caminho que comece pelo texto antes do `*`. Os caminhos são relativos à raiz da aplicação, sem a query string.

> **Aviso** As rotas excluídas têm de autenticar os pedidos de outra forma, por exemplo com uma assinatura de webhook.

## Regenerar o token

Regenere tanto o ID da sessão como o token quando o utilizador inicia ou termina sessão:

```php
session()->regenerate();
app('csrf')->regenerate();
```

## Personalização

O `Luany\Framework\Http\Middleware\CsrfMiddleware` pode ser estendido ainda mais:

| Membro | Finalidade |
| --- | --- |
| `$except` | Caminhos excluídos |
| `$failureStatus` | Código de status em caso de falha |
| `shouldVerify(Request $request)` | Decide se um pedido é verificado |
| `tokenFromRequest(Request $request)` | De onde é lido o token |

O serviço de token é o `Luany\Framework\Security\CsrfToken`, registado como `csrf`:

| Método | Descrição |
| --- | --- |
| `token()` | Token atual, gerado na primeira utilização (64 caracteres hexadecimais) |
| `validate(string $token)` | Comparação em tempo constante com o token guardado |
| `regenerate()` | Guarda e devolve um token novo |
