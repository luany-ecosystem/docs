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

Locales outside `supported_locales` are rejected: `setLocale()` ignores them. List the default locale there too — if it is missing, the translator starts with the fallback locale when that one is supported, otherwise with the first supported locale.

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
3. the default locale — `config('app.locale')`, then the `APP_LOCALE` environment variable, the same setting the translator uses;
4. `'en'`, when it is supported — otherwise the translator keeps its starting locale.

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

## Regional locales

A regional locale falls back to its language, then to the fallback locale: with `APP_LOCALE=pt_AO`, a key is looked up in `lang/pt_AO.php`, then `lang/pt.php`, then `lang/en.php`. Put only what differs in the regional file.

Validation messages follow the same chain. The framework ships them in Portuguese (`pt`, with `pt_BR` overriding "ficheiro" → "arquivo"); any line can be overridden in your `lang/{locale}.php` with the same key, for example `'validation.required' => 'Preencha :field.'`.

## Dates

`format_date()` writes a date in the current locale:

```lte
{{ format_date($post->created_at) }}            {{-- 23/11/2026 --}}
{{ format_date($post->created_at, 'long') }}    {{-- 23 de novembro de 2026 --}}
```

| Style | Portuguese | English |
|---|---|---|
| `short` (default) | `23/11/2026` | `2026-11-23` |
| `medium` | `23 nov 2026` | `Nov 23, 2026` |
| `long` | `23 de novembro de 2026` | `November 23, 2026` |
| `full` | `segunda-feira, 23 de novembro de 2026` | `Monday, November 23, 2026` |
| `time` | `14:05` | `14:05` |
| `datetime` | `23/11/2026 14:05` | `2026-11-23 14:05` |

Dates are shown in the app timezone (`APP_TIMEZONE`). Portuguese and English do not need the intl extension; other locales use it when it is installed.

## Long-running servers

Under PHP-FPM every request starts clean. On long-running servers (Swoole, RoadRunner, FrankenPHP workers) the translator is shared between requests: register `LocaleMiddleware` so each request sets its own locale — otherwise every request uses the default locale.

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
