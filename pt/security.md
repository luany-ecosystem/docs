# Segurança

O Luany é seguro por omissão sempre que possível: o output é escapado, os pedidos que alteram estado precisam de um token CSRF, os modelos recusam a atribuição em massa até que a permita e os identificadores SQL são validados. Esta página resume as proteções incluídas, o que tem de configurar por si e uma checklist para produção.

## Proteções incluídas

| Ameaça | Proteção | Predefinição | Detalhes |
|---|---|---|---|
| Cross-site scripting (XSS) | `{{ }}` escapa o output | Ativa | [Sintaxe dos templates](lte-syntax.md) |
| Cross-site request forgery | `CsrfMiddleware` verifica `POST`, `PUT`, `PATCH`, `DELETE` | Ativa no skeleton | [Proteção CSRF](csrf.md) |
| Atribuição em massa | Os modelos estão totalmente protegidos até `$fillable` ser definido | Ativa | [Modelos](models.md) |
| Injeção de SQL | Os valores são passados por binding; os identificadores são validados e delimitados | Ativa | [Query builder](query-builder.md) |
| Sequestro / fixação de sessão | Cookie de sessão reforçado, strict mode, `regenerate()` | Ativa | [Sessão e mensagens flash](session.md) |
| Redirecionamentos abertos | A validação só redireciona para referers do mesmo host | Ativa | [Validação](validation.md) |
| Falsificação de IP / HTTPS | Os cabeçalhos forwarded são ignorados, a menos que o proxy seja de confiança | Ativa | [Middleware](middleware.md#proxies-de-confiança) |
| Fuga de erros | Páginas de debug só com `APP_DEBUG=true` | Depende do `.env` | [Tratamento de erros](errors.md) |
| Pedidos cross-origin | `CorsMiddleware` | Opcional | [Middleware](middleware.md#cors) |
| Força bruta | `RateLimitMiddleware` | Opcional | [Middleware](middleware.md#rate-limiting) |
| Dotfiles e listagens expostos | `public/.htaccess` bloqueia dotfiles e índices | Só Apache | [Deploy](deployment.md) |

## Escape do output

`{{ $value }}` é escapado com `htmlspecialchars()` usando `ENT_QUOTES | ENT_SUBSTITUTE` e UTF-8, pelo que as aspas são seguras dentro de atributos e UTF-8 inválido não consegue apagar o output. `{!! $value !!}` imprime HTML em bruto.

```lte
<p>{{ $comment->body }}</p>          {{-- escapado --}}
<div>{!! $page->trustedHtml !!}</div> {{-- em bruto: só para HTML que controla --}}
<script>const user = @json($user);</script>
```

> **Aviso** Nunca use `{!! !!}` com dados introduzidos pelo utilizador. Use `@json` para passar dados ao JavaScript — codifica os caracteres com significado em HTML. Em código PHP, `e($value)` escapa da mesma forma que `{{ }}`.

As diretivas próprias devolvem código que **não** é escapado; faça o escape dentro do código gerado (consulte [Diretivas próprias](lte-custom.md)).

## CSRF

O skeleton regista `App\Http\Middleware\CsrfMiddleware` globalmente. Todos os pedidos `POST`, `PUT`, `PATCH` e `DELETE` têm de levar o token da sessão no campo `_token` (`@csrf`) ou no cabeçalho `X-CSRF-TOKEN`; os tokens são comparados com `hash_equals()`. Uma discrepância devolve `419` (em JSON para clientes JSON).

```lte
<form method="POST" action="/profile">
    @csrf
    @method('PUT')
    ...
</form>
```

Exclua apenas endpoints que se autentiquem de outra forma, como webhooks assinados, com `$except = ['/webhooks/*']`.

## Atribuição em massa

Um modelo com `$fillable` vazio está totalmente protegido (`$guarded = ['*']`). `fill()` e `update()` descartam silenciosamente todos os atributos que não sejam fillable, e `create()` lança uma exceção quando não resta nada fillable — chaves arbitrárias do pedido nunca chegam à base de dados. Liste as colunas que os utilizadores podem definir:

```php
class User extends Model
{
    protected array $fillable = ['name', 'email', 'password'];
    protected array $hidden   = ['password'];
}
```

Combine-o com a validação — `validate()` devolve apenas os campos validados, pelo que `User::create($data)` nunca vê dados a mais, como `is_admin`.

## Injeção de SQL

O query builder faz binding de todos os valores. Os nomes de tabelas e colunas que chegam ao SQL são validados contra um padrão estrito de identificadores e delimitados conforme o driver, e os operadores são verificados contra uma whitelist — nomes inválidos lançam `InvalidArgumentException` em vez de serem concatenados. As regras de validação `unique` e `exists` validam e delimitam os seus nomes de tabela e coluna da mesma forma.

> **Aviso** O SQL em bruto que escreve por si (`$pdo->query()`, expressões raw) não está protegido. Use sempre prepared statements com parâmetros por binding.

## Sessões

A sessão em ficheiros arranca com definições de cookie seguras:

| Opção | Predefinição |
|---|---|
| `httponly` | `true` — sem acesso a partir de JavaScript |
| `samesite` | `Lax` |
| `secure` | Automático em HTTPS (`$_SERVER['HTTPS']` ou porta 443) |
| `strict_mode` | `true` — IDs de sessão não inicializados são rejeitados |
| `only_cookies` | `true` — sem IDs de sessão nos URLs |

Regenere o ID de sessão e o token CSRF quando os privilégios do utilizador mudam (login, logout):

```php
session()->regenerate();
app('csrf')->regenerate();
```

> **Nota** Atrás de um proxy que termina o TLS, o PHP não vê HTTPS, pelo que a flag `secure` automática fica desligada. Defina `'secure' => true` em `config/session.php`.

## Proxies de confiança

`$request->ip()` e `$request->isSecure()` ignoram `X-Forwarded-For` e `X-Forwarded-Proto`, a menos que `REMOTE_ADDR` seja um proxy de confiança, para que os clientes não possam falsificar o seu endereço (o que anularia o rate limiting). Atrás de um load balancer, declare-o explicitamente como de confiança:

```php
use Luany\Core\Http\Request;

Request::setTrustedProxies(['10.0.0.0/8']);
```

## Redirecionamentos seguros

Quando `validate()` falha sem um URL de redirecionamento explícito, redireciona para o `Referer` apenas se este for um caminho relativo ou apontar para o host atual; qualquer outra coisa (`//evil.com`, `https://evil.com`, URLs `javascript:`, credenciais no URL) recorre a `/`. Os dados antigos devolvidos aos formulários como mensagens flash nunca contêm `password*`, `_token`, `csrf_token` nem `_method`.

Quando redirecionar por si para um URL retirado do pedido, valide-o da mesma forma:

```php
use Luany\Framework\Support\Url;

$next   = $request->input('next');
$target = Url::sameHostOrFallback(is_string($next) ? $next : null, (string) $request->header('Host', ''));

return redirect($target);
```

## CORS e rate limiting

Ambos são middleware opcional do `luany/core`:

```php
use Luany\Core\Middleware\CorsMiddleware;
use Luany\Core\Middleware\RateLimitMiddleware;
use Luany\Core\RateLimit\FileRateLimiter;

Route::post('/login', [AuthController::class, 'login'])
    ->middleware(new RateLimitMiddleware(
        new FileRateLimiter(base_path('storage/rate-limits')),
        maxAttempts: 5,
        decaySeconds: 60,
    ));

Route::middleware(new CorsMiddleware(
    allowedOrigins:   ['https://app.example.com'],
    allowCredentials: true,
))->group(function () {
    Route::get('/api/me', [ApiController::class, 'me']);
});
```

O `CorsMiddleware` recusa a combinação insegura de `['*']` com `allowCredentials: true`. Os pedidos limitados recebem `429` com `Retry-After`.

## Upload de ficheiros

O validador não tem regras para ficheiros (`file`, `image`, `mimes`). Valide os uploads por si: verifique o código `error`, limite o tamanho, detete o tipo MIME a partir do conteúdo, gere o nome do ficheiro guardado e guarde os ficheiros fora de `public/`, a menos que se destinem a ser públicos.

```php
$file = $request->file('avatar');

if ($file === null || $file['error'] !== UPLOAD_ERR_OK || $file['size'] > 2 * 1024 * 1024) {
    abort(422, 'Invalid upload.');
}

$mime = (new \finfo(FILEINFO_MIME_TYPE))->file($file['tmp_name']);
if (!in_array($mime, ['image/png', 'image/jpeg'], true)) {
    abort(422, 'Only PNG and JPEG images are allowed.');
}

$name = bin2hex(random_bytes(16)) . ($mime === 'image/png' ? '.png' : '.jpg');
move_uploaded_file($file['tmp_name'], base_path('storage/uploads/' . $name));
```

## Checklist

- [ ] `APP_ENV=production` e `APP_DEBUG=false` em produção.
- [ ] `APP_KEY` definida e `.env` sem permissão de leitura para os outros utilizadores do servidor web (`chmod 640`).
- [ ] A raiz dos documentos do servidor web é `public/` — nunca a raiz do projeto.
- [ ] O `CsrfMiddleware` está no Kernel; `$except` contém apenas webhooks assinados.
- [ ] Todos os modelos definem `$fillable`; as colunas sensíveis estão em `$hidden`.
- [ ] Nenhum `{!! !!}` com dados controlados pelo utilizador.
- [ ] ID de sessão e token CSRF regenerados no login e no logout.
- [ ] HTTPS em todo o lado; cookie de sessão `secure` e HSTS ativados.
- [ ] Proxies de confiança configurados quando atrás de um load balancer.
- [ ] Rate limiting nas rotas de login, registo e reposição de palavra-passe.
- [ ] Uploads validados e guardados fora de `public/`.
- [ ] Dependências atualizadas (`composer audit`).

Consulte [Deploy](deployment.md) para o lado do servidor.
