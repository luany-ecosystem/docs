# Localization

Luany's translator loads strings from PHP files in `lang/` and picks the active locale per request. It has no external dependencies.

## Translation files

Create one file per locale in `lang/` at the project root. Each returns a **flat** array — the dots in keys are part of the key name, not nesting:

```php
// lang/en.php
return [
    'nav.home'         => 'Home',
    'nav.posts'        => 'Posts',
    'welcome.title'    => 'Welcome, :name!',
    'footer.copyright' => '© :year :name',
];
```

```php
// lang/pt.php
return [
    'nav.home'      => 'Início',
    'nav.posts'     => 'Artigos',
    'welcome.title' => 'Bem-vindo, :name!',
];
```

## Translating strings

```php
__('nav.home');                                     // 'Home'
__('welcome.title', ['name' => 'Ana']);             // 'Welcome, Ana!'
__('footer.copyright', ['year' => date('Y'), 'name' => 'Luany']);
```

In LTE views:

```lte
<html lang="{{ locale() }}">
...
<a href="{{ route('home') }}">{{ __('nav.home') }}</a>
```

Lookup order: the active locale, then the fallback locale, then the **key itself** — a missing translation shows `nav.home` rather than an empty string. Placeholders are written `:name` in the string and passed without the colon.

`locale()` returns the active locale code.

## Configuration

The translator reads `config/app.php`, falling back to environment variables:

| Setting | Config key | Environment | Default |
| --- | --- | --- | --- |
| Default locale | `app.locale` | `APP_LOCALE` | `'en'` |
| Fallback locale | `app.fallback_locale` | `APP_FALLBACK_LOCALE` | `'en'` |
| Supported locales | `app.supported_locales` | `APP_SUPPORTED_LOCALES` (comma-separated) | `[locale, fallback]` |

The skeleton's `config/app.php` lists only English. Add every locale you ship:

```php
'locale'            => env('APP_LOCALE', 'en'),
'fallback_locale'   => 'en',
'supported_locales' => ['en', 'pt'],
```

> **Warning** The default locale must be in `supported_locales`. Locales outside the list are rejected, and an unsupported default leaves the translator without an active locale.

## Detecting the locale per request

Add `LocaleMiddleware` to the global middleware in `app/Http/Kernel.php`:

```php
use Luany\Framework\Http\Middleware\LocaleMiddleware;

protected array $middleware = [
    LiveReloadMiddleware::class,
    CsrfMiddleware::class,
    LocaleMiddleware::class,
];
```

On each request it chooses the first supported locale from:

1. the `app_locale` cookie — the user's explicit choice;
2. the `Accept-Language` header — the browser preference, by quality (`pt-PT,pt;q=0.9,en;q=0.8` → `pt`; entries with `q=0` are ignored);
3. the `APP_LOCALE` environment variable;
4. `'en'`.

Only the primary language subtag is used: `pt-BR` and `pt-PT` both select `pt`.

## Letting users switch language

Set the `app_locale` cookie and send the user back. Redirect only to a same-host `Referer` to avoid an open redirect:

```php
use Luany\Core\Http\Request;
use Luany\Core\Routing\Route;
use Luany\Framework\Support\Url;

Route::get('/locale/{locale}', function (Request $request, string $locale) {
    if (!app('translator')->isSupported($locale)) {
        abort(404);
    }

    $back = Url::sameHostOrFallback($request->header('Referer'), $request->header('Host'));

    return redirect($back)->cookie('app_locale', $locale, time() + 60 * 60 * 24 * 365, secure: $request->isSecure());
})->name('locale');
```

```lte
<a href="{{ route('locale', ['locale' => 'pt']) }}">Português</a>
```

## The translator

The translator is `Luany\Framework\Support\Translator`, bound as `translator`:

| Method | Description |
| --- | --- |
| `get($key, $replace = [])` | Translate (what `__()` calls) |
| `setLocale($locale)` | Change the active locale; unsupported locales are ignored |
| `getLocale()` | Active locale |
| `getFallback()` | Fallback locale |
| `getSupported()` | Supported locales |
| `isSupported($locale)` | Whether a locale is allowed |

```php
app('translator')->setLocale('pt');
```

Each locale file is loaded once, on first use. The translator does not handle pluralization — use separate keys such as `cart.item` and `cart.items`.
