# Sessão e mensagens flash

As sessões guardam dados entre pedidos do mesmo visitante. O driver predefinido do Luany guarda-as em ficheiros, usando a gestão de sessões nativa do PHP, com definições de cookie reforçadas.

## Usar a sessão

O helper `session()` devolve a sessão ou lê um valor:

```php
session()->set('cart_id', 42);

$cartId = session('cart_id');            // 42
$theme  = session('theme', 'light');     // valor predefinido quando não existe

session()->has('cart_id');               // true
session()->forget('cart_id');
```

A sessão está registada no container como `session` e como `Luany\Framework\Contracts\SessionInterface`, pelo que também pode ser injetada:

```php
use Luany\Framework\Contracts\SessionInterface;

public function show(SessionInterface $session): string
{
    return (string) $session->get('cart_id');
}
```

A sessão é iniciada na primeira utilização — os pedidos que nunca lhe tocam não criam nenhuma. Os ficheiros de sessão são escritos em `storage/sessions` (criada com permissões `0700`).

### Métodos

| Método | Descrição |
| --- | --- |
| `get($key, $default = null)` | Lê um valor (incluindo dados flash) |
| `set($key, $value)` | Guarda um valor |
| `has($key)` | Indica se um valor existe |
| `forget($key)` | Remove um valor |
| `all()` | Todos os valores, incluindo os dados flash atuais |
| `flash($key, $value)` | Guarda um valor apenas para o pedido seguinte |
| `regenerate()` | Novo ID de sessão; o ficheiro da sessão antiga é apagado |
| `getId()` | O ID da sessão |
| `save()` | Escreve e fecha a sessão antecipadamente |
| `destroy()` | Apaga a sessão e os respetivos dados |

## Dados flash

Os valores em flash estão disponíveis apenas no pedido **seguinte** e são depois removidos — ideais para mensagens de estado após um redirecionamento:

```php
session()->flash('status', 'Profile updated.');
return redirect('/profile');
```

```php
// pedido seguinte
session('status');   // 'Profile updated.'
// o pedido a seguir a esse: null
```

### Mensagens flash

Para o caso habitual de uma mensagem por redirecionamento, use `flash()` e `get_flash()`:

```php
flash('success', 'Post created.');
return redirect('/posts');
```

`get_flash()` devolve `['type' => 'success', 'message' => 'Post created.']`, ou `null`. O layout do skeleton inclui `views/components/flash.lte`, que a apresenta:

```lte
@php($flash = get_flash())
@if($flash)
    <div class="flash flash-{{ $flash['type'] }}" role="status">{{ $flash['message'] }}</div>
@endif
```

O tipo é texto livre — `success`, `error`, `warning` e `info` são escolhas comuns. Chamar `flash()` duas vezes no mesmo pedido mantém apenas a última mensagem.

### Dados antigos

Quando a [validação](validation.md) falha, o `validate()` guarda em flash os dados submetidos. O `old()` lê-os de volta para voltar a preencher o formulário:

```lte
<input name="email" value="{{ old('email') }}">
<input name="name" value="{{ old('name', $user->name) }}">
```

## Configuração

Crie o `config/session.php` para alterar as predefinições. Todas as chaves são opcionais:

```php
return [
    'name'     => 'myapp_session',
    'lifetime' => 0,          // segundos; 0 = até o browser ser fechado
    'secure'   => null,       // null = automático em HTTPS
    'samesite' => 'Lax',
];
```

| Opção | Predefinição | Descrição |
| --- | --- | --- |
| `save_path` | `storage/sessions` | Pasta dos ficheiros de sessão |
| `name` | `session.name` do PHP | Nome do cookie |
| `lifetime` | `0` | Duração do cookie em segundos |
| `path` | `'/'` | Caminho do cookie |
| `domain` | `''` | Domínio do cookie (`''` = apenas o host atual) |
| `secure` | `null` | `true` / `false`, ou `null` para ativar automaticamente em HTTPS |
| `httponly` | `true` | Esconde o cookie do JavaScript |
| `samesite` | `'Lax'` | `'Lax'`, `'Strict'` ou `'None'` |
| `strict_mode` | `true` | `session.use_strict_mode` — rejeita IDs de sessão desconhecidos |
| `only_cookies` | `true` | `session.use_only_cookies`, e desativa `use_trans_sid` |

> **Nota** A deteção automática de `secure` só confia na perspetiva do próprio servidor (`HTTPS`, porta 443). Atrás de um proxy ou load balancer que termina o TLS, defina `'secure' => true`.

## Início e fim de sessão

Regenere o ID da sessão — e o [token CSRF](csrf.md) — sempre que os privilégios do utilizador mudarem, para evitar a fixação de sessão (session fixation):

```php
public function login(Request $request): Response
{
    $data = validate($request->body(), ['email' => 'required|email', 'password' => 'required']);

    // Pressupõe uma coluna "password" com hash na tabela users.
    $user = User::firstWhere('email = ?', [$data['email']]);
    if (!$user || !password_verify($data['password'], $user->password)) {
        flash('error', 'Invalid credentials.');
        return redirect('/login');
    }

    session()->regenerate();
    app('csrf')->regenerate();
    session()->set('user_id', $user->id);

    return redirect('/dashboard');
}

public function logout(): Response
{
    session()->forget('user_id');
    session()->regenerate();
    app('csrf')->regenerate();

    return redirect('/');
}
```

## Drivers personalizados

Para guardar as sessões noutro sítio (Redis, uma base de dados), implemente `Luany\Framework\Contracts\SessionInterface` e associe-a no método `register()` de um provider. O Kernel deixa então de registar a sua predefinição:

```php
$app->singleton('session', function () {
    $session = new \App\Session\RedisSession(new \Redis()); // a sua implementação de SessionInterface
    $session->start();
    return $session;
});
```

Inicie a sessão na factory, como mostrado — os helpers esperam uma sessão já iniciada.
