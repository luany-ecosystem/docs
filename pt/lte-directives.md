# Referência de diretivas

Todas as diretivas incorporadas do LTE, agrupadas por finalidade. A lista corresponde a `Luany\Lte\Compiler::BUILTIN_DIRECTIVES`; qualquer outro `@palavra` é texto simples, a menos que o [registe](lte-custom.md).

"Args" assinala as diretivas que exigem argumentos — chamá-las sem eles é um [erro de compilação](lte-errors.md).

## Condicionais

| Diretiva | Args | Compila para | Ramos |
| --- | --- | --- | --- |
| `@if(cond)` … `@endif` | sim | `if (cond):` | `@elseif(cond)`, `@else` |
| `@unless(cond)` … `@endunless` | sim | `if (!(cond)):` | `@else` |
| `@isset(vars)` … `@endisset` | sim | `if (isset(vars)):` | `@else` |
| `@ifempty(expr)` … `@endifempty` | sim | `if (empty(expr)):` | `@else` |
| `@empty(expr)` … `@endempty` | sim | `if (empty(expr)):` | `@else` |

`@else` tem de ser o último ramo; um segundo `@else` ou um `@elseif` depois dele é rejeitado.

## Ciclos

| Diretiva | Args | Notas |
| --- | --- | --- |
| `@foreach(expr as $v)` … `@endforeach` | sim | `foreach` normal. |
| `@for(init; cond; step)` … `@endfor` | sim | `for` normal. |
| `@while(cond)` … `@endwhile` | sim | `while` normal. |
| `@forelse(expr as $v)` … `@empty` … `@endforelse` | sim | `@empty` (sem argumentos, no máximo uma vez) é renderizado quando não houve nada para iterar. Funciona com `Traversable`s vazios e generators. `@empty` é opcional. |

`@forelse` exige a palavra-chave `as`; `@forelse($items)` é um erro de compilação.

## Layouts, includes e componentes

| Diretiva | Args | Comportamento |
| --- | --- | --- |
| `@extends('layout')` | sim | Define o layout-pai. Só é respeitada na página renderizada e nos seus layouts. |
| `@section('name')` … `@endsection` | — | Captura uma secção. |
| `@section('name')` … `@stop` | — | `@stop` é um alias de `@endsection`. |
| `@section('name', $value)` | — | Secção inline; `$value` é escapado. |
| `@yield('name', $default = '')` | sim | Imprime uma secção; o valor predefinido é escapado. |
| `@include('view', $data = [])` | sim | Renderiza uma vista com as variáveis da vista-mãe juntas a `$data`. |
| `@component('view', $data = [])` … `@endcomponent` | sim | Renderiza um componente; o conteúdo interior passa a ser `$slot` (um `HtmlString`). |
| `@slot('name')` … `@endslot` | sim | Slot nomeado dentro de `@component`, exposto como `$name` (um `HtmlString`). |

Os argumentos de nome podem ser strings entre aspas, nomes com pontos sem aspas (`@extends(layouts.main)`) ou qualquer expressão PHP (`@include($view)`). Ver [Layouts e componentes](lte-layouts.md).

## Assets e stacks

| Diretiva | Args | Comportamento |
| --- | --- | --- |
| `@style` … `@endstyle` | — | Recolhe um bloco `<style>` (sem duplicados). |
| `@script` / `@script(module, defer, async)` … `@endscript` | — | Recolhe um bloco `<script>`; `module` acrescenta `type="module"`, `defer` / `async` acrescentam esses atributos. |
| `@styles` | — | Marcador substituído por todos os estilos recolhidos, no fim da renderização completa. |
| `@scripts` | — | Marcador substituído por todos os scripts recolhidos, no fim da renderização completa. |
| `@push('name')` … `@endpush` | sim | Acrescenta conteúdo a uma stack nomeada. |
| `@stack('name')` | sim | Imprime o conteúdo de uma stack. |

Detalhes em [Assets e stacks](lte-assets.md).

## Formulários

### `@csrf`

Gera o campo CSRF oculto usado pela [proteção CSRF](csrf.md) do framework:

```lte
<form method="POST" action="/posts">
    @csrf
    ...
</form>
```

```html
<input type="hidden" name="_token" value="3f9a…">
```

O valor vem do helper `csrf_token()` quando este existe (sempre, numa aplicação Luany); caso contrário, o LTE guarda um token aleatório em `$_SESSION['_csrf_token']`.

### `@method`

Os formulários HTML só enviam GET e POST. `@method` acrescenta o campo de substituição `_method`, em maiúsculas:

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

Um literal (`'put'` ou `PUT`) é resolvido em tempo de compilação; uma expressão (`@method($verb)`) é escapada em tempo de execução.

## Guardas de autenticação

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

Ambas chamam `Luany\Lte\Runtime::check()`. Por predefinição, um visitante está autenticado quando `$_SESSION['user_id']` está definido. Ligue a sua própria lógica uma única vez, por exemplo em `AppServiceProvider::boot()`:

```php
use Luany\Lte\Runtime;

public function boot(Application $app): void
{
    Runtime::setAuthResolver(fn (): bool => session('auth_user_id') !== null);
}
```

| Método do `Runtime` | Descrição |
| --- | --- |
| `setAuthResolver(?callable $resolver)` | Callable que devolve `bool`; `null` repõe o comportamento predefinido baseado na sessão. |
| `check(): bool` | Usado por `@auth` / `@guest`. |
| `setDebug(bool $debug)` / `debug(): bool` | Ativa `@dump` / `@dd`. |
| `reset()` | Repõe os valores predefinidos (debug ativo, autenticação baseada na sessão). |

## Saída de dados

### `@json`

Codifica um valor como JSON seguro para incorporar em HTML e em tags `<script>`:

```lte
<script>
    const config = @json($config);
    const users  = @json($users, JSON_PRETTY_PRINT);
</script>
```

As flags `JSON_HEX_TAG | JSON_HEX_AMP | JSON_HEX_APOS | JSON_HEX_QUOT | JSON_UNESCAPED_UNICODE` são sempre aplicadas, pelo que `</script>` é emitido como `<\/script>`. As flags adicionais passadas no segundo argumento são combinadas com estas por OR. O resultado não é envolvido em aspas nem em `<pre>`.

### `@class`

Constrói um atributo `class` a partir de um array condicional. As entradas com chave inteira são sempre incluídas; as entradas com chave string só quando o seu valor é verdadeiro:

```lte
<button @class(['btn', 'btn-primary' => $isPrimary, 'btn-disabled' => !$active])>
    Save
</button>
```

```html
<button class="btn btn-primary">
```

A lista de classes é escapada. A mesma lógica está disponível em PHP através de `Luany\Lte\ClassHelper::compile(array $classes): string` (string de classes) e de `ClassHelper::attr(array $classes): string` (atributo `class="…"` completo).

## PHP e depuração

| Diretiva | Args | Comportamento |
| --- | --- | --- |
| `@php(statement)` | — | Executa uma instrução: `@php($count = count($items))`. |
| `@php` … `@endphp` | — | Bloco de PHP em bruto; a sintaxe LTE no interior não é interpretada. |
| `@dump(expr)` | sim | `var_dump()` — só quando `Runtime::debug()` é verdadeiro. |
| `@dd(expr)` | sim | `var_dump()` seguido de `die` — só quando `Runtime::debug()` é verdadeiro. |

O framework chama `Runtime::setDebug()` com `config('app.debug')` (recorrendo a `APP_DEBUG` como alternativa), por isso `@dump` e `@dd` ficam silenciosos em produção.

## Saída e escape (não são diretivas)

| Sintaxe | Comportamento |
| --- | --- |
| `{{ expr }}` | Output com escape. |
| `{!! expr !!}` | Output sem escape. |
| `{{-- … --}}` | Comentário, removido em tempo de compilação. |
| `@@` / `@{{ … }}` | `@` literal / `{{ … }}` literal. |

Ver [Sintaxe dos templates](lte-syntax.md).
