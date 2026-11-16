# CSRF Protection

Cross-site request forgery tricks a logged-in user's browser into submitting a request to your app. Luany prevents it by requiring a secret per-session token on every state-changing request.

## How it works

The skeleton registers `App\Http\Middleware\CsrfMiddleware` as global middleware. For every `POST`, `PUT`, `PATCH` and `DELETE` request it compares the submitted token with the one stored in the session. `GET`, `HEAD` and `OPTIONS` requests are never checked.

| | Name |
| --- | --- |
| Form field | `_token` (the legacy `csrf_token` field is also accepted) |
| HTTP header | `X-CSRF-TOKEN` |
| Session key | `_csrf_token` |

The token is read from the first non-empty string in that order — form field, legacy field, header — and compared in constant time.

The token stays the same for the whole session, so several tabs, the back button and concurrent AJAX requests keep working. It is not rotated on each request.

## Forms

Add the token to every form that does not use `GET`:

```lte
<form method="POST" action="/posts">
    @csrf
    <input name="title">
    <button>Save</button>
</form>
```

`@csrf` renders a hidden `_token` input. Outside LTE, use the helpers:

```php
<?= csrf_field() ?>        // <input type="hidden" name="_token" value="...">
<?= csrf_token() ?>        // the token itself
```

## JavaScript requests

The skeleton layout publishes the token in a meta tag:

```lte
<meta name="csrf-token" content="{{ csrf_token() }}">
```

Send it in the `X-CSRF-TOKEN` header:

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

## Failed verification

| Request | Response |
| --- | --- |
| Expects JSON | `{"message": "CSRF token mismatch."}` |
| Otherwise | Plain text `CSRF token mismatch.` |

The status is the middleware's `$failureStatus`: **419 Page Expired** in the skeleton (the framework base class defaults to 403). A missing, empty or non-string token (for example `_token[]=x`) is a mismatch, never a server error.

A mismatch usually means the session expired or the form was rendered before the session was regenerated — reload the page to get a fresh token.

## Excluding URIs

Third-party webhooks cannot know your token. List their paths in `$except`:

```php
class CsrfMiddleware extends BaseCsrfMiddleware
{
    protected array $except = [
        '/stripe/webhook',     // exact path
        '/webhooks/*',         // any path starting with /webhooks/
    ];

    protected int $failureStatus = 419;
}
```

A pattern matches the exact path, or — with a trailing `*` — any path that starts with the text before the `*`. Paths are relative to the application root, without the query string.

> **Warning** Excluded routes must authenticate requests some other way, such as a webhook signature.

## Regenerating the token

Regenerate both the session ID and the token when the user logs in or out:

```php
session()->regenerate();
app('csrf')->regenerate();
```

## Customizing

`Luany\Framework\Http\Middleware\CsrfMiddleware` can be extended further:

| Member | Purpose |
| --- | --- |
| `$except` | Excluded paths |
| `$failureStatus` | Status code on mismatch |
| `shouldVerify(Request $request)` | Decide whether a request is checked |
| `tokenFromRequest(Request $request)` | Where the token is read from |

The token service is `Luany\Framework\Security\CsrfToken`, bound as `csrf`:

| Method | Description |
| --- | --- |
| `token()` | Current token, generated on first use (64 hex characters) |
| `validate(string $token)` | Constant-time comparison with the stored token |
| `regenerate()` | Store and return a new token |
