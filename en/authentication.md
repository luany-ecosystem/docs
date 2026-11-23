# Authentication

Session authentication ships with luany/framework 1.9+: log users in, protect routes, reset passwords and verify email addresses.

## Setup

The guard reads users from a luany/database model — `App\Models\User` by default — with an `email` and a hashed `password` column. Another model goes in `config/auth.php`:

```php
return [
    'model'         => App\Models\User::class,
    'login'         => '/login',               // where guests are sent
    'home'          => '/dashboard',           // where logged-in users are sent from /login
    'verify_notice' => '/email/verify-notice', // where unverified users are sent
];
```

Hash passwords with `app('hash')` (Argon2id when PHP has it, bcrypt otherwise):

```php
User::create(['email' => $email, 'password' => app('hash')->make($password)]);
```

## Logging in and out

```php
public function login(Request $request): Response
{
    $data = validate($request->body(), ['email' => 'required|email', 'password' => 'required|string']);

    if (!auth()->attempt($data)) {
        session()->flash('errors', ['email' => ['These credentials do not match our records.']]);
        return redirect('/login');
    }

    return redirect(auth()->intended('/dashboard'));
}

public function logout(Request $request): Response
{
    auth()->logout();
    return redirect('/');
}
```

| Method | Returns |
|---|---|
| `auth()->attempt($credentials)` | `true` and logs in when they match |
| `auth()->user()` / `auth()->id()` | the logged-in user / its id, or `null` |
| `auth()->check()` / `auth()->guest()` | whether someone is logged in |
| `auth()->login($user)` / `auth()->logout()` | log in a user you already trust / log out |
| `auth()->intended($default)` | the page a guest tried to open before logging in |

On login and logout the session id is regenerated and the CSRF token replaced. An unknown email takes as long to check as a wrong password, so the response time does not reveal which emails exist. Old password hashes are upgraded automatically on login.

Limit login attempts with the rate limiter middleware of luany/core on the `POST /login` route.

## Protecting routes

```php
use Luany\Framework\Http\Middleware\Authenticate;
use Luany\Framework\Http\Middleware\EnsureEmailIsVerified;
use Luany\Framework\Http\Middleware\RedirectIfAuthenticated;

Route::middleware(RedirectIfAuthenticated::class)->group(function () {
    Route::get('/login', [LoginController::class, 'show']);
});

Route::middleware([Authenticate::class, EnsureEmailIsVerified::class])->group(function () {
    Route::get('/dashboard', [DashboardController::class, 'index']);
});
```

`Authenticate` answers `401` to JSON requests and redirects other guests to `/login`, remembering the page. In views, `@auth … @endauth` and `@guest … @endguest` follow the same state.

## Password reset

Create the token table once (in a migration):

```php
$pdo->exec(\Luany\Framework\Auth\DatabaseTokenRepository::createTableSql());
```

```php
// POST /forgot-password
$email  = (string) $request->input('email');
$status = app('auth.passwords')->sendResetLink(['email' => $email], $request->ip(),
    function (object $user, string $token) use ($email): void {
        mailer()->to($email)->subject('Reset your password')->send('emails.reset-password', [
            'url' => url('/reset-password?token=' . $token . '&email=' . urlencode($email)),
        ]);
    });
// Show the same message for every email: "If the address exists, a link is on its way."
// $status is PasswordBroker::THROTTLED after too many requests.

// POST /reset-password
$status = app('auth.passwords')->reset([
    'email' => $request->input('email'), 'token' => $request->input('token'), 'password' => $newPassword,
]);
// PasswordBroker::PASSWORD_RESET or PasswordBroker::INVALID_TOKEN
```

When the account exists, the token and the email are made **after** the response is sent (PHP-FPM), so the response takes the same time whether or not the email exists. Without PHP-FPM (for example `php -S`) the client may wait for the email to be sent.

Behind a load balancer or CDN, set `TRUSTED_PROXIES` in `.env` (IPs or CIDR ranges, comma-separated) so the per-IP limit counts the real client; otherwise everyone behind the proxy shares it. `X-Forwarded-For` from any other address is ignored.

Tokens are stored only as a SHA-256 hash, work once, expire after 60 minutes, and stop working when a newer link is requested or the password changes in any other way. Requests are limited to 3 per email and 10 per IP every 15 minutes.

## Email verification

```php
// after sign-up
$url = url(app('auth.verification')->url($user->id, $user->email));
mailer()->to($user->email)->subject('Confirm your email')->send('emails.verify', ['url' => $url]);

// GET /email/verify — behind ValidateSignature
Route::get('/email/verify', [VerifyEmailController::class, 'verify'])->middleware(ValidateSignature::class);

public function verify(Request $request): Response
{
    $user = User::find((int) $request->query('id')) ?? abort(404);
    if (!app('auth.verification')->verify($request, $user->id, $user->email)) {
        abort(403);
    }
    $user->update(['email_verified_at' => date('Y-m-d H:i:s')]);
    return redirect('/dashboard');
}
```

Links are signed with the `APP_KEY`, expire after one hour and stop working if the user changes email. `ValidateSignature` rejects modified or expired links and sends `Referrer-Policy: no-referrer`, so the link does not leak to other sites. **Rotating the `APP_KEY` invalidates every verification link already sent.**

`app('url.signer')` signs any other URL the same way: `->sign('/download', ['file' => $id], expiresIn: 600)`.
