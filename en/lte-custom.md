# Custom Directives

Register your own `@directives` on the compiler. A directive is a compile-time function: it receives the raw argument string and returns the PHP (or HTML) that replaces it in the compiled template.

## Registering a directive

Register directives in a service provider's `boot()` method, before any view is rendered:

```php
namespace App\Providers;

use Luany\Framework\Application;
use Luany\Framework\ServiceProvider;

class AppServiceProvider extends ServiceProvider
{
    public function boot(Application $app): void
    {
        $compiler = $app->make('view')->getCompiler();

        $compiler->directive('money', function (?string $args): string {
            return "<?php echo \\Luany\\Lte\\Escaper::escape(number_format((float) ({$args}), 2, '.', ',')); ?>";
        });

        $compiler->directive('datetime', function (?string $args): string {
            $format = $args ?: "'Y-m-d H:i'";
            return "<?php echo date({$format}); ?>";
        });
    }
}
```

```lte
<p>Price: @money($product->price)</p>
<p>Generated at @datetime('d/m/Y H:i')</p>
```

The signature is `Compiler::directive(string $name, callable $handler): void`. Outside the framework, call it on `$engine->getCompiler()`.

## The handler contract

| | |
| --- | --- |
| **Input** | The text between the parentheses, trimmed, or `null` when the directive has none (`@hr`). |
| **Output** | A string inserted verbatim into the compiled PHP file. |
| **When** | At compile time — once per template, not per render. Runtime values must be read inside the generated `<?php … ?>` code. |
| **Names** | Letters, digits and underscores. A name that matches a built-in replaces the built-in. |

Arguments are scanned string- and bracket-aware, so `@money(max($a, $b))` and `@icon('a)b')` arrive intact.

> **Warning** The returned code is not escaped. Escape output yourself — `\Luany\Lte\Escaper::escape($value)` applies the same rules as `{{ }}` — and never place user input into the generated code at compile time.

## Block directives

Return opening and closing fragments from two directives:

```php
$compiler->directive('admin', fn (): string =>
    "<?php if (\\Luany\\Lte\\Runtime::check() && (session('role') === 'admin')): ?>");

$compiler->directive('endadmin', fn (): string => '<?php endif; ?>');
```

```lte
@admin
    <a href="/admin">Admin panel</a>
@endadmin
```

Because LTE does not know their structure, custom directives are not validated: a missing `@endadmin` produces invalid PHP that fails when the view is rendered, not at compile time. Keep blocks small and test them.

> **Note** Like built-in closers, a registered directive whose name starts with `end` is recognised right after text (`Done@endadmin`). Any other custom directive directly after a letter, digit or underscore is text (`x@money(1)`), so separate it with a space.

## Recognition and caching

- Only registered names become directives. Before registration — or in a view compiled without the provider — `@money(…)` is plain text.
- The set of registered directive names is part of the compiled-cache key, so registering or removing a directive automatically recompiles every view. Changing the *body* of an existing handler does not change the key: run `luany cache:clear` (or `Engine::clearCache()`) after editing a handler.
- `Compiler::isDirective(string $name): bool` reports whether a name is built-in or registered.

## Examples

A translated string, escaped:

```php
$compiler->directive('t', fn (?string $args): string =>
    "<?php echo \\Luany\\Lte\\Escaper::escape(__({$args})); ?>");
```

```lte
<h1>@t('nav.home')</h1>
<p>@t('footer.copyright', ['year' => date('Y')])</p>
```

A feature flag block:

```php
$compiler->directive('feature', fn (?string $args): string =>
    "<?php if (config('features.' . {$args}, false)): ?>");
$compiler->directive('endfeature', fn (): string => '<?php endif; ?>');
```

```lte
@feature('new-checkout')
    @include('checkout.v2')
@endfeature
```
