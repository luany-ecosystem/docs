# Template Errors

LTE validates every template when it compiles it, before any PHP runs. Mistakes such as an unclosed `@foreach` or a typo inside `{{ }}` are reported with the view name, line and column. This page explains what is checked, how errors look, how runtime errors map back to `.lte` lines, and how the compiled cache works.

## The compile pipeline

```
.lte source → Parser (tokens with line/column)
            → Ast\TreeBuilder (syntax tree + validation)
            → Compiler (PHP) → cache file → evaluate → HTML
```

The parser is hand-written (no regular expressions) and string- and bracket-aware, so `{{ '}}' }}` and `@include('card', ['a' => 1, 'b' => 2])` parse correctly.

## What is validated

| Check | Example error |
| --- | --- |
| Every block is closed | `Unclosed @foreach: missing @endforeach (line 12, column 5).` |
| Closers match the innermost block | `Unexpected @endif: @foreach opened on line 3 is still open (expected @endforeach) (line 4, column 1).` |
| No stray closers | `Unexpected @endsection: there is no open block to close (line 9, column 1).` |
| Branches sit in the right block | `Unexpected @else inside @foreach (line 12) (line 15, column 3).` |
| `@else` is the last branch | `Unexpected @elseif: @else (line 6) must be the last branch of @if (line 8, column 1).` |
| Bare `@empty` only once, inside `@forelse` | `Duplicate @empty in @forelse (line 10, column 1).` |
| Required arguments are present | `@include requires arguments, e.g. @include(…) (line 3, column 1).` |
| Embedded PHP is syntactically valid | `PHP syntax error in {{ }} on line 2, column 4: syntax error, unexpected token ")".` |

PHP is checked in `{{ }}`, `{!! !!}`, `@php` blocks and directive arguments using PHP's own parser (`token_get_all()` with `TOKEN_PARSE`). Nothing is executed. Unclosed tags (`{{` without `}}`, `{{--` without `--}}`, `@php` without `@endphp`) are reported by the parser.

[Custom directives](lte-custom.md) are not validated: LTE does not know their arguments or structure.

When the engine compiles a view, the message is prefixed with the view name:

```
LTE compilation error in view [pages.users]: Unclosed @foreach: missing @endforeach (line 12, column 5).
```

### Glued closers

A closer must be separated from the text after it. `Title@endslotBody` is one unknown word (`endslotBody`), so it stays text and the slot stays open. LTE detects this and adds a hint:

```lte
@component('components.card')
    @slot('title')Title@endslotBody
@endcomponent
```

```
Unexpected @endcomponent: @slot opened on line 2 is still open (expected @endslot)
— "@endslotBody" on line 2 is read as text: separate the directive from the text ("@endslot Body") (line 3, column 1).
```

Text directly *before* a closer is fine: `Title@endslot` works.

## `TemplateSyntaxException`

Compile errors throw `Luany\Lte\TemplateSyntaxException`, which extends `\RuntimeException` (existing `catch (\RuntimeException)` blocks keep working). It exposes:

| Property | Type | Description |
| --- | --- | --- |
| `$view` | `?string` | Dot-notation view name, when known. |
| `$templateLine` | `int` | 1-based line in the `.lte` file (`0` = unknown). |
| `$templateColumn` | `int` | 1-based column, in bytes (`0` = unknown). |

```php
use Luany\Lte\TemplateSyntaxException;

try {
    $html = view('pages.report', $data);
} catch (TemplateSyntaxException $e) {
    logger('Broken template', [
        'view'   => $e->view,
        'line'   => $e->templateLine,
        'column' => $e->templateColumn,
    ], 'error');
    throw $e;
}
```

The same exception is thrown at render time for structural problems the compiler cannot see, such as a circular `@extends` chain or a view that leaves an output buffer open.

## Checking every view

Run the validator over the whole `views/` directory without rendering anything — ideal for CI:

```bash
luany view:check
```

See [view:check](view-check.md) for options and GitHub Actions annotations.

### Inspecting the syntax tree

Tooling can parse and validate one view with `Engine::parseView()`. It throws `TemplateSyntaxException` (message prefixed `In view [name]:`) or `\RuntimeException` when the view does not exist, and otherwise returns an `Ast\Template`:

```php
use Luany\Lte\Ast\BlockNode;
use Luany\Lte\Ast\Node;

$tree = app('view')->parseView('pages.home');

$tree->walk(function (Node $node, int $depth): void {
    if ($node instanceof BlockNode) {
        echo str_repeat('  ', $depth) . "@{$node->name} (line {$node->line})\n";
    }
});
```

| Node | Description |
| --- | --- |
| `Ast\Template` | Root. `$children`, `walk(callable $visitor)` (depth-first, blocks before their children), `tokens()`. |
| `Ast\BlockNode` | A block directive: `$name`, `$branches` (list of `Ast\Branch`), `$closing`, `opening()`. |
| `Ast\Branch` | One branch (`@if`, `@elseif`, `@else`, `@empty`, …): `$directive` and `$children`. |
| `Ast\TokenNode` | Leaf: text, echo, raw echo, PHP block or a non-block directive. `type()`, `name()`, `$token`. |

Every node has `$line` and `$column`. The compiler API used by the engine is also public: `Compiler::buildTree(array $tokens)`, `Compiler::compileTree(Template $tree)` and `Compiler::validate(array $tokens)`.

## Runtime errors

Exceptions thrown while a compiled view runs (an undefined method, a `TypeError`) are rethrown as `\RuntimeException` with the original `.lte` file and line; the original exception is available through `getPrevious()`:

```
LTE render error in [home.lte line 23]: Call to undefined method App\Models\User::missing()
```

The compiler embeds `@lte:N` line markers in the generated PHP, and the engine maps the failing PHP line back to the nearest marker. If a render fails half-way, open output buffers and section, stack and component state are unwound, so long-running workers start the next request clean.

In development, the framework's [error page](errors.md) shows these messages; in production they are logged.

## The compiled cache

Compiled templates are plain PHP files named by a SHA-256 hash.

| Aspect | Behaviour |
| --- | --- |
| Location | `storage/cache/views` in a Luany app. A standalone `Engine` without `$cachePath` uses a private per-user directory (`sys_get_temp_dir()/lte_cache_<uid>`, mode `0700`) and refuses one that is a symlink or owned by another user. |
| Cache key | Engine version (`Engine::VERSION`, currently `1.2.0`) + registered custom directive names + view path. Upgrading LTE or registering a directive never reuses stale code. |
| Freshness | With `autoReload` off, a view recompiles when its source is at least as new as its compiled file (same-second edits are detected). With `autoReload` on (`APP_DEBUG=true` in the framework), every render recompiles. |
| Atomic writes | Each compile writes a temporary file and `rename()`s it into place, so concurrent requests never read a half-written file. OPcache is invalidated for the new file. |

Clear the cache after deploying, or after changing a custom directive's handler:

```bash
luany cache:clear
```

`Engine::clearCache()` does the same from PHP.
