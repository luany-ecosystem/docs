# Directives Reference

Every built-in LTE directive, grouped by purpose. The list matches `Luany\Lte\Compiler::BUILTIN_DIRECTIVES`; any other `@word` is plain text unless you [register it](lte-custom.md).

"Args" marks directives that require arguments — calling them without is a [compile error](lte-errors.md).

## Conditionals

| Directive | Args | Compiles to | Branches |
| --- | --- | --- | --- |
| `@if(cond)` … `@endif` | yes | `if (cond):` | `@elseif(cond)`, `@else` |
| `@unless(cond)` … `@endunless` | yes | `if (!(cond)):` | `@else` |
| `@isset(vars)` … `@endisset` | yes | `if (isset(vars)):` | `@else` |
| `@ifempty(expr)` … `@endifempty` | yes | `if (empty(expr)):` | `@else` |
| `@empty(expr)` … `@endempty` | yes | `if (empty(expr)):` | `@else` |

`@else` must be the last branch; a second `@else` or an `@elseif` after it is rejected.

## Loops

| Directive | Args | Notes |
| --- | --- | --- |
| `@foreach(expr as $v)` … `@endforeach` | yes | Standard `foreach`. |
| `@for(init; cond; step)` … `@endfor` | yes | Standard `for`. |
| `@while(cond)` … `@endwhile` | yes | Standard `while`. |
| `@forelse(expr as $v)` … `@empty` … `@endforelse` | yes | `@empty` (no arguments, at most once) renders when nothing was iterated. Works with empty `Traversable`s and generators. `@empty` is optional. |

`@forelse` requires the `as` keyword; `@forelse($items)` is a compile error.

## Layouts, includes and components

| Directive | Args | Behaviour |
| --- | --- | --- |
| `@extends('layout')` | yes | Sets the parent layout. Honoured only for the rendered page and its layouts. |
| `@section('name')` … `@endsection` | — | Captures a section. |
| `@section('name')` … `@stop` | — | `@stop` is an alias of `@endsection`. |
| `@section('name', $value)` | — | Inline section; `$value` is HTML-escaped. |
| `@yield('name', $default = '')` | yes | Prints a section; the default is HTML-escaped. |
| `@include('view', $data = [])` | yes | Renders a view with the parent's variables merged with `$data`. |
| `@component('view', $data = [])` … `@endcomponent` | yes | Renders a component; inner content becomes `$slot` (an `HtmlString`). |
| `@slot('name')` … `@endslot` | yes | Named slot inside `@component`, exposed as `$name` (an `HtmlString`). |

Name arguments may be quoted strings, bare dotted names (`@extends(layouts.main)`) or any PHP expression (`@include($view)`). See [Layouts & Components](lte-layouts.md).

## Assets and stacks

| Directive | Args | Behaviour |
| --- | --- | --- |
| `@style` … `@endstyle` | — | Collects a `<style>` block (deduplicated). |
| `@script` / `@script(module, defer, async)` … `@endscript` | — | Collects a `<script>` block; `module` adds `type="module"`, `defer` / `async` add those attributes. |
| `@styles` | — | Placeholder replaced by all collected styles after the full render. |
| `@scripts` | — | Placeholder replaced by all collected scripts after the full render. |
| `@push('name')` … `@endpush` | yes | Appends content to a named stack. |
| `@stack('name')` | yes | Prints a stack's content. |

Details in [Assets & Stacks](lte-assets.md).

## Forms

### `@csrf`

Outputs the hidden CSRF field used by the framework's [CSRF protection](csrf.md):

```lte
<form method="POST" action="/posts">
    @csrf
    ...
</form>
```

```html
<input type="hidden" name="_token" value="3f9a…">
```

The value comes from the `csrf_token()` helper when it exists (always, in a Luany app); otherwise LTE stores a random token in `$_SESSION['_csrf_token']`.

### `@method`

HTML forms only send GET and POST. `@method` adds the `_method` override field, upper-cased:

```lte
<form method="POST" action="/posts/{{ $post->id }}">
    @csrf
    @method('DELETE')
    <button>Delete</button>
</form>
```

```html
<input type="hidden" name="_method" value="DELETE">
```

A literal (`'put'` or `PUT`) is resolved at compile time; an expression (`@method($verb)`) is escaped at runtime.

## Authentication guards

```lte
@auth
    <a href="/account">My account</a>
@else
    <a href="/login">Sign in</a>
@endauth

@guest
    <p>Create an account to save your progress.</p>
@endguest
```

Both call `Luany\Lte\Runtime::check()`. By default a visitor is authenticated when `$_SESSION['user_id']` is set. Plug in your own logic once, for example in `AppServiceProvider::boot()`:

```php
use Luany\Lte\Runtime;

public function boot(Application $app): void
{
    Runtime::setAuthResolver(fn (): bool => session('auth_user_id') !== null);
}
```

| `Runtime` method | Description |
| --- | --- |
| `setAuthResolver(?callable $resolver)` | Callable returning `bool`; `null` restores the session default. |
| `check(): bool` | Used by `@auth` / `@guest`. |
| `setDebug(bool $debug)` / `debug(): bool` | Enables `@dump` / `@dd`. |
| `reset()` | Restores defaults (debug on, session-based auth). |

## Data output

### `@json`

Encodes a value as JSON that is safe to embed in HTML and `<script>` tags:

```lte
<script>
    const config = @json($config);
    const users  = @json($users, JSON_PRETTY_PRINT);
</script>
```

The flags `JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_UNESCAPED_UNICODE` are always applied, so `</script>` is emitted as `<\/script>`. Extra flags in the second argument are OR-ed with them. The output is not wrapped in quotes or `<pre>`.

### `@class`

Builds a `class` attribute from a conditional array. Integer-keyed entries are always included; string-keyed entries only when their value is truthy:

```lte
<button @class(['btn', 'btn-primary' => $isPrimary, 'btn-disabled' => !$active])>
    Save
</button>
```

```html
<button class="btn btn-primary">
```

The class list is HTML-escaped. The same logic is available in PHP through `Luany\Lte\ClassHelper::compile(array $classes): string` (class string) and `ClassHelper::attr(array $classes): string` (full `class="…"` attribute).

## PHP and debugging

| Directive | Args | Behaviour |
| --- | --- | --- |
| `@php(statement)` | — | Runs one statement: `@php($count = count($items))`. |
| `@php` … `@endphp` | — | Raw PHP block; LTE syntax inside is not parsed. |
| `@dump(expr)` | yes | `var_dump()` — only when `Runtime::debug()` is true. |
| `@dd(expr)` | yes | `var_dump()` then `die` — only when `Runtime::debug()` is true. |

The framework calls `Runtime::setDebug()` with `config('app.debug')` (falling back to `APP_DEBUG`), so `@dump` and `@dd` are silent in production.

## Output and escaping (not directives)

| Syntax | Behaviour |
| --- | --- |
| `{{ expr }}` | Escaped output. |
| `{!! expr !!}` | Raw output. |
| `{{-- … --}}` | Comment, removed at compile time. |
| `@@` / `@{{ … }}` | Literal `@` / literal `{{ … }}`. |

See [Template Syntax](lte-syntax.md).
