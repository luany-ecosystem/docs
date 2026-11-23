# Autenticação

A autenticação por sessão vem com o luany/framework 1.9+: iniciar sessão, proteger rotas, recuperar palavras-passe e verificar emails.

## Configuração

A guarda lê os utilizadores de um modelo do luany/database — `App\Models\User` por omissão — com uma coluna `email` e uma coluna `password` com o hash. Outro modelo define-se em `config/auth.php`:

```php
return [
    'model'         => App\Models\User::class,
    'login'         => '/login',               // para onde vão os visitantes
    'home'          => '/painel',              // para onde vai quem já tem sessão, vindo de /login
    'verify_notice' => '/email/verify-notice', // para onde vão os utilizadores por verificar
];
```

Faça o hash das palavras-passe com `app('hash')` (Argon2id quando o PHP o tem, bcrypt caso contrário):

```php
User::create(['email' => $email, 'password' => app('hash')->make($senha)]);
```

## Entrar e sair

```php
public function login(Request $request): Response
{
    $dados = validate($request->body(), ['email' => 'obrigatorio|email', 'password' => 'obrigatorio|texto']);

    if (!auth()->attempt($dados)) {
        session()->flash('errors', ['email' => ['Estes dados não correspondem a nenhuma conta.']]);
        return redirect('/login');
    }

    return redirect(auth()->intended('/painel'));
}

public function logout(Request $request): Response
{
    auth()->logout();
    return redirect('/');
}
```

| Método | Devolve |
|---|---|
| `auth()->attempt($credenciais)` | `true` e inicia a sessão quando correspondem |
| `auth()->user()` / `auth()->id()` | o utilizador com sessão / o seu id, ou `null` |
| `auth()->check()` / `auth()->guest()` | se há alguém com sessão |
| `auth()->login($user)` / `auth()->logout()` | iniciar sessão com um utilizador já confirmado / terminar a sessão |
| `auth()->intended($padrao)` | a página que o visitante tentou abrir antes de entrar |

No login e no logout o id da sessão é regenerado e o token CSRF substituído. Um email desconhecido demora tanto a verificar como uma palavra-passe errada, por isso o tempo de resposta não revela que emails existem. Os hashes antigos são atualizados automaticamente no login.

Limite as tentativas de login com o middleware de rate limit do luany/core na rota `POST /login`.

## Proteger rotas

```php
use Luany\Framework\Http\Middleware\Authenticate;
use Luany\Framework\Http\Middleware\EnsureEmailIsVerified;
use Luany\Framework\Http\Middleware\RedirectIfAuthenticated;

Route::middleware(RedirectIfAuthenticated::class)->group(function () {
    Route::get('/login', [LoginController::class, 'show']);
});

Route::middleware([Authenticate::class, EnsureEmailIsVerified::class])->group(function () {
    Route::get('/painel', [PainelController::class, 'index']);
});
```

O `Authenticate` responde `401` a pedidos JSON e manda os outros visitantes para `/login`, lembrando a página. Nas vistas, `@auth … @endauth` e `@guest … @endguest` (ou `@autenticado` e `@visitante`) seguem o mesmo estado.

## Recuperar a palavra-passe

Crie a tabela dos tokens uma vez (numa migração):

```php
$pdo->exec(\Luany\Framework\Auth\DatabaseTokenRepository::createTableSql());
```

```php
// POST /forgot-password
$email  = (string) $request->input('email');
$estado = app('auth.passwords')->sendResetLink(['email' => $email], $request->ip(),
    function (object $user, string $token) use ($email): void {
        mailer()->to($email)->subject('Redefinir a palavra-passe')->send('emails.redefinir', [
            'url' => url('/reset-password?token=' . $token . '&email=' . urlencode($email)),
        ]);
    });
// Mostre a mesma mensagem para qualquer email: "Se o endereço existir, o link vai a caminho."
// $estado é PasswordBroker::THROTTLED depois de pedidos a mais.

// POST /reset-password
$estado = app('auth.passwords')->reset([
    'email' => $request->input('email'), 'token' => $request->input('token'), 'password' => $novaSenha,
]);
// PasswordBroker::PASSWORD_RESET ou PasswordBroker::INVALID_TOKEN
```

Os tokens ficam guardados só como hash SHA-256, servem uma vez, expiram ao fim de 60 minutos e deixam de valer quando se pede um link novo ou a palavra-passe muda por outra via. Os pedidos estão limitados a 3 por email e 10 por IP a cada 15 minutos.

## Verificação de email

```php
// depois do registo
$url = url(app('auth.verification')->url($user->id, $user->email));
mailer()->to($user->email)->subject('Confirme o seu email')->send('emails.verificar', ['url' => $url]);

// GET /email/verify — com ValidateSignature
Route::get('/email/verify', [VerificarEmailController::class, 'verify'])->middleware(ValidateSignature::class);

public function verify(Request $request): Response
{
    $user = User::find((int) $request->query('id')) ?? abort(404);
    if (!app('auth.verification')->verify($request, $user->id, $user->email)) {
        abort(403);
    }
    $user->update(['email_verified_at' => date('Y-m-d H:i:s')]);
    return redirect('/painel');
}
```

Os links são assinados com a `APP_KEY`, expiram ao fim de uma hora e deixam de valer se o utilizador mudar de email. O `ValidateSignature` rejeita links alterados ou expirados e envia `Referrer-Policy: no-referrer`, para o link não passar a outros sites. **Rodar a `APP_KEY` invalida todos os links de verificação já enviados.**

O `app('url.signer')` assina qualquer outro URL da mesma forma: `->sign('/download', ['file' => $id], expiresIn: 600)`.
