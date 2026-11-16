# Template Syntax

LTE (Luany Template Engine) compiles `.lte` files to plain PHP and caches the result. This page covers rendering views, echoing data, escaping, comments, control structures and embedded PHP.

## Rendering views

Views live in `views/` and use the `.lte` extension. Names use dot notation: `pages.home` resolves to `views/pages/home.lte` (a `.php` file with the same name is accepted as a fallback).

Render a view with the `view()` helper and pass data as an array; each key becomes a variable:

```php
use Luany\Core\Http\Request;

class HomeController extends Controller
{
    public function index(Request $request): string
    {
        return view('pages.home', ['title' => 'Welcome', 'user' => $user]);
    }
}
```

The framework registers the engine as `app('view')` with views in `views/`, compiled files in `storage/cache/views` and automatic recompilation when `APP_DEBUG=true`. Outside the framework, create the engine yourself:

```php
use Luany\Lte\Engine;

$engine = new Engine(
    viewsPath:  __DIR__ . '/views',
    cachePath:  __DIR__ . '/storage/cache/views',
    autoReload: true,   // recompile on every render (development)
);

echo $engine->render('pages.home', ['title' => 'Welcome']);
```

| Engine method | Description |
| --- | --- |
| `render(string $view, array $data = []): string` | Compile (if needed), evaluate and return HTML. |
| `getCompiler(): Compiler` | Access the compiler, e.g. to [register directives](lte-custom.md). |
| `parseView(string $view): Ast\Template` | Parse and validate without rendering (see [Template Errors](lte-errors.md#inspecting-the-syntax-tree)). |
| `findView(string $view): ?string` | Absolute path of a view, or `null`. |
| `clearCache(): void` | Delete every compiled file. |
| `setAutoReload(bool $autoReload): void` | Toggle recompilation on every render. |
| `getCachePath(): string` | The compiled-template directory. |

## Displaying data

Wrap an expression in `{{ }}` to print it HTML-escaped:

```lte
<h1>{{ $title }}</h1>
<p>{{ $user->email }}</p>
<p>{{ strtoupper($user->name) }} has {{ count($posts) }} posts.</p>
```

Any PHP expression is allowed. Output goes through `Luany\Lte\Escaper::escape()`, which:

- uses `htmlspecialchars()` with `ENT_QUOTES | ENT_SUBSTITUTE` and UTF-8, so quotes are encoded and invalid UTF-8 becomes U+FFFD instead of an empty string;
- prints `null` as an empty string;
- prints an `Htmlable` value as-is (see below).

The expression is compiled as `($expression) ?? ''`, so an undefined variable prints nothing instead of raising a warning.

### Raw output

`{!! !!}` prints the value without escaping:

```lte
<div class="post-body">{!! $post->html !!}</div>
```

> **Warning** Never use `{!! !!}` with user-supplied content. It is the classic XSS hole.

### Trusted HTML: `Htmlable` and `HtmlString`

An object implementing `Luany\Lte\Htmlable` (one method, `toHtml(): string`) is never escaped by `{{ }}`, inline `@section` values or `@yield` defaults. `Luany\Lte\HtmlString` wraps a string you trust:

```php
use Luany\Lte\HtmlString;

return view('pages.about', [
    'badge' => new HtmlString('<span class="badge">New</span>'),
]);
```

```lte
<h2>About {{ $badge }}</h2>
```

Implement `Htmlable` on your own value objects (a Markdown renderer, a pagination widget) to let them render their markup through `{{ }}`.

Component [slots](lte-layouts.md#components) are `HtmlString`s too, so `{{ $slot }}` prints them unescaped. `HtmlString::isEmpty()` / `isNotEmpty()` tell whether the wrapped HTML is empty or only whitespace.

## Comments

`{{-- --}}` comments are removed at compile time and never reach the HTML:

```lte
{{-- Rendered only for admins --}}
```

## Escaping LTE syntax

| You write | Rendered as | Use case |
| --- | --- | --- |
| `@@if` | `@if` | Show a directive name literally. |
| `@@` | `@` | A lone at-sign next to a directive name. |
| `@{{ message }}` | `{{ message }}` | Client-side frameworks (Vue, Alpine). |

Other `@` signs need no escaping:

- **Unknown words are text.** Only built-in and [registered](lte-custom.md) directives are parsed, so `@media`, `@keyframes` and `@username` stay as written.
- **`word@word` is text.** An `@` directly after a letter, digit or underscore is not a directive, so `me@if.com` and `dev@php.net` are safe. Closing directives (`@end…`, `@else`, `@elseif`, `@stop`) are the exception — `Saved@endif` still closes the block.

```lte
<style>@media (max-width: 600px) { .nav { display: none; } }</style>
<p>Contact support@luany.dev or type @{{ name }} in the template.</p>
```

## Control structures

### Conditionals

```lte
@if($user->isAdmin())
    <a href="/admin">Dashboard</a>
@elseif($user->isEditor())
    <a href="/editor">Editor</a>
@else
    <a href="/profile">Profile</a>
@endif

@unless($user->verified)
    <p>Please verify your email.</p>
@endunless
```

`@isset`, `@ifempty` and `@empty($x)` test a variable; all accept an `@else` branch:

```lte
@isset($subtitle)
    <h2>{{ $subtitle }}</h2>
@endisset

@ifempty($cart)
    <p>Your cart is empty.</p>
@else
    <p>{{ count($cart) }} items</p>
@endifempty
```

`@empty($x) … @endempty` is equivalent to `@ifempty`. Without arguments, `@empty` is the empty branch of `@forelse`.

### Loops

```lte
@foreach($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

@for($i = 1; $i <= 3; $i++)
    <span>Step {{ $i }}</span>
@endfor

@while($item = array_shift($queue))
    <p>{{ $item }}</p>
@endwhile
```

`@forelse` adds an empty branch. It works with arrays, `Traversable`s and generators, and may be nested:

```lte
@forelse($posts as $post)
    <article>{{ $post->title }}</article>
@empty
    <p>No posts yet.</p>
@endforelse
```

`@empty` is optional — `@forelse … @endforelse` alone behaves like `@foreach`.

> **Note** LTE has no `@break`, `@continue` or `@switch`. Use `@php(break)` / `@php(continue)` inside a loop, or `@if` / `@elseif` chains. Register your own with [custom directives](lte-custom.md).

Whitespace is allowed between a directive and its arguments: `@if ($x)` works like `@if($x)`.

## Embedded PHP

Inline form, for a single statement:

```lte
@php($total = array_sum(array_column($items, 'price')))
<p>Total: {{ number_format($total, 2) }}</p>
```

Block form, closed by `@endphp`. Its content is copied verbatim — LTE syntax inside it is not parsed:

```lte
@php
    $grouped = [];
    foreach ($items as $item) {
        $grouped[$item->category][] = $item;
    }
@endphp
```

> **Tip** Keep logic in controllers. `@php` is fine for small view-local values; anything larger belongs in PHP classes.

## Next steps

- [Layouts & Components](lte-layouts.md) — `@extends`, `@section`, `@include`, `@component`.
- [Directives Reference](lte-directives.md) — every built-in directive.
- [Template Errors](lte-errors.md) — compile-time validation and caching.
