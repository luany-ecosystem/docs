# Session & Flash

Sessions keep data between requests for the same visitor. Luany's default driver stores them in files using PHP's native session handling, with hardened cookie settings.

## Using the session

The `session()` helper returns the session, or reads a value:

```php
session()->set('cart_id', 42);

$cartId = session('cart_id');            // 42
$theme  = session('theme', 'light');     // default when missing

session()->has('cart_id');               // true
session()->forget('cart_id');
```

The session is bound in the container as `session` and as `Luany\Framework\Contracts\SessionInterface`, so it can also be injected:

```php
use Luany\Framework\Contracts\SessionInterface;

public function show(SessionInterface $session): string
{
    return (string) $session->get('cart_id');
}
```

It starts on first use — requests that never touch the session do not create one. Session files are written to `storage/sessions` (created with `0700` permissions), which the skeleton's `.gitignore` excludes — see [Security](security.md#sessions).

### Methods

| Method | Description |
| --- | --- |
| `get($key, $default = null)` | Read a value (flash data included) |
| `set($key, $value)` | Store a value |
| `has($key)` | Whether a value exists |
| `forget($key)` | Remove a value |
| `all()` | All values, including current flash data |
| `flash($key, $value)` | Store a value for the next request only |
| `regenerate()` | New session ID; the old session file is deleted |
| `getId()` | The session ID |
| `save()` | Write and close the session early |
| `destroy()` | Delete the session and its data |

## Flash data

Flashed values are available on the **next** request only, then removed — ideal for status messages after a redirect:

```php
session()->flash('status', 'Profile updated.');
return redirect('/profile');
```

```php
// next request
session('status');   // 'Profile updated.'
// the request after that: null
```

### Flash messages

For the usual one-message-per-redirect case, use `flash()` and `get_flash()`:

```php
flash('success', 'Post created.');
return redirect('/posts');
```

`get_flash()` returns `['type' => 'success', 'message' => 'Post created.']`, or `null`. The skeleton's layout includes `views/components/flash.lte`, which renders it:

```lte
@php($flash = get_flash())
@if($flash)
    <div class="flash flash-{{ $flash['type'] }}" role="status">{{ $flash['message'] }}</div>
@endif
```

The type is free text — `success`, `error`, `warning` and `info` are common choices. Calling `flash()` twice in one request keeps only the last message.

### Old input

When [validation](validation.md) fails, `validate()` flashes the submitted input. `old()` reads it back to refill the form:

```lte
<input name="email" value="{{ old('email') }}">
<input name="name" value="{{ old('name', $user->name) }}">
```

## Configuration

Create `config/session.php` to change the defaults. Every key is optional:

```php
return [
    'name'     => 'myapp_session',
    'lifetime' => 0,          // seconds; 0 = until the browser closes
    'secure'   => null,       // null = automatic on HTTPS
    'samesite' => 'Lax',
];
```

| Option | Default | Description |
| --- | --- | --- |
| `save_path` | `storage/sessions` | Directory for session files |
| `name` | PHP's `session.name` | Cookie name |
| `lifetime` | `0` | Cookie lifetime in seconds |
| `path` | `'/'` | Cookie path |
| `domain` | `''` | Cookie domain (`''` = current host only) |
| `secure` | `null` | `true` / `false`, or `null` to enable automatically on HTTPS |
| `httponly` | `true` | Hide the cookie from JavaScript |
| `samesite` | `'Lax'` | `'Lax'`, `'Strict'` or `'None'` |
| `strict_mode` | `true` | `session.use_strict_mode` — reject unknown session IDs |
| `only_cookies` | `true` | `session.use_only_cookies`, and disables `use_trans_sid` |

> **Note** Automatic `secure` detection only trusts the server's own view (`HTTPS`, port 443). Behind a proxy or load balancer that terminates TLS, set `'secure' => true`.

## Login and logout

Regenerate the session ID — and the [CSRF token](csrf.md) — whenever the user's privileges change, to prevent session fixation:

```php
public function login(Request $request): Response
{
    $data = validate($request->body(), ['email' => 'required|email', 'password' => 'required']);

    // Assumes a hashed "password" column on the users table.
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

## Custom drivers

To store sessions elsewhere (Redis, a database), implement `Luany\Framework\Contracts\SessionInterface` and bind it in a provider's `register()` method. The Kernel then leaves its default alone:

```php
$app->singleton('session', function () {
    $session = new \App\Session\RedisSession(new \Redis()); // your SessionInterface implementation
    $session->start();
    return $session;
});
```

Start the session in the factory as shown — the helpers expect a started session.
