# Diretivas próprias

Registe as suas próprias `@diretivas` no compilador. Uma diretiva é uma função de tempo de compilação: recebe a string de argumentos em bruto e devolve o PHP (ou HTML) que a substitui no template compilado.

## Registar uma diretiva

Registe as diretivas no método `boot()` de um service provider, antes de qualquer vista ser renderizada:

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

A assinatura é `Compiler::directive(string $name, callable $handler): void`. Fora do framework, chame-a em `$engine->getCompiler()`.

## O contrato do handler

| | |
| --- | --- |
| **Entrada** | O texto entre parênteses, sem espaços nas pontas, ou `null` quando a diretiva não tem parênteses (`@hr`). |
| **Saída** | Uma string inserida tal como está no ficheiro PHP compilado. |
| **Quando** | Em tempo de compilação — uma vez por template, não por renderização. Os valores de tempo de execução têm de ser lidos dentro do código `<?php … ?>` gerado. |
| **Nomes** | Letras, dígitos e underscores. Um nome igual ao de uma diretiva incorporada substitui-a. |

Os argumentos são analisados tendo em conta strings e parênteses, por isso `@money(max($a, $b))` e `@icon('a)b')` chegam intactos.

> **Aviso** O código devolvido não é escapado. Faça o escape do output por sua conta — `\Luany\Lte\Escaper::escape($value)` aplica as mesmas regras que `{{ }}` — e nunca coloque input do utilizador no código gerado em tempo de compilação.

## Diretivas de bloco

Devolva os fragmentos de abertura e de fecho a partir de duas diretivas:

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

Como o LTE não conhece a sua estrutura, as diretivas próprias não são validadas: um `@endadmin` em falta produz PHP inválido, que falha quando a vista é renderizada e não em tempo de compilação. Mantenha os blocos pequenos e teste-os.

> **Nota** Tal como as diretivas de fecho incorporadas, uma diretiva registada cujo nome começa por `end` é reconhecida logo a seguir a texto (`Done@endadmin`). Qualquer outra diretiva própria imediatamente a seguir a uma letra, dígito ou underscore é texto (`x@money(1)`), por isso separe-a com um espaço.

## Reconhecimento e cache

- Só os nomes registados se tornam diretivas. Antes do registo — ou numa vista compilada sem o provider — `@money(…)` é texto simples.
- O conjunto dos nomes de diretivas registadas faz parte da chave da cache de compilação, por isso registar ou remover uma diretiva recompila automaticamente todas as vistas. Alterar o *corpo* de um handler existente não muda a chave: execute `luany cache:clear` (ou `Engine::clearCache()`) depois de editar um handler.
- `Compiler::isDirective(string $name): bool` indica se um nome é incorporado ou registado.

## Exemplos

Uma string traduzida, com escape:

```php
$compiler->directive('t', fn (?string $args): string =>
    "<?php echo \\Luany\\Lte\\Escaper::escape(__({$args})); ?>");
```

```lte
<h1>@t('nav.home')</h1>
<p>@t('footer.copyright', ['year' => date('Y')])</p>
```

Um bloco de feature flag:

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
