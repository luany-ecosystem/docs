# Portuguese Directives

LTE can be written with Portuguese directive names. `@se … @senao … @fimse` compiles to exactly the same PHP as `@if … @else … @endif`. The English names keep working, and both can be mixed in one template.

Requires luany/lte 1.4 and luany/framework 1.7.

## Turning it on

The syntax is a project setting. In `config/app.php`:

```php
'syntax' => env('LUANY_SYNTAX', 'en'),   // 'pt' adds the Portuguese names
```

and in `.env`:

```dotenv
LUANY_SYNTAX=pt
```

Outside a Luany app, call `$engine->setSyntax('pt')` on the LTE engine.

It is opt-in on purpose: in LTE a `@word` that is not a directive stays text. With `pt`, words such as `@se` or `@para` become directives, which could change a page that shows them as text.

## Example

```lte
@estende('layouts.main')

@secao('conteudo')
    @listar($produtos as $produto)
        @se($produto->ativo)
            <li>{{ $produto->nome }}</li>
        @senao
            <li class="inativo">{{ $produto->nome }}</li>
        @fimse
    @vazio
        <p>Sem produtos.</p>
    @fimlistar
@fimsecao
```

## Reference

| English | Portuguese | English | Portuguese |
|---|---|---|---|
| `@if` / `@endif` | `@se` / `@fimse` | `@extends` | `@estende` |
| `@elseif` | `@senaose` | `@section` / `@endsection` | `@secao` / `@fimsecao` |
| `@else` | `@senao` | `@yield` | `@exibir` |
| `@unless` / `@endunless` | `@exceto` / `@fimexceto` | `@include` | `@incluir` |
| `@isset` / `@endisset` | `@definido` / `@fimdefinido` | `@component` / `@endcomponent` | `@componente` / `@fimcomponente` |
| `@ifempty` / `@endifempty` | `@sevazio` / `@fimsevazio` | `@method` | `@metodo` |
| `@empty($x)` / `@endempty` | `@vazio($x)` / `@fimvazio` | `@auth` / `@endauth` | `@autenticado` / `@fimautenticado` |
| `@foreach` / `@endforeach` | `@paracada` / `@fimparacada` | `@guest` / `@endguest` | `@visitante` / `@fimvisitante` |
| `@forelse` … `@empty` / `@endforelse` | `@listar` … `@vazio` / `@fimlistar` | `@class` | `@classe` |
| `@for` / `@endfor` | `@para` / `@fimpara` | | |
| `@while` / `@endwhile` | `@enquanto` / `@fimenquanto` | | |

## Rules

- **Accents and spelling variants are accepted:** `@senão`, `@senãose`, `@método`; `@seccao`, `@seção` and `@secção` all mean `@secao`.
- **Lowercase only,** like the English directives: `@Se` is text.
- **Technical names have no Portuguese form:** `@php`, `@csrf`, `@json`, `@style`, `@styles`, `@script`, `@scripts`, `@push`, `@stack`, `@slot`, `@dump`, `@dd`.
- **Errors use the name you wrote:** `Unexpected @fimse: @paracada opened on line 12 is still open (expected @fimparacada)`. `luany view:check` and the VS Code extension follow the project's syntax.
- **[Custom directives](lte-custom.md) win:** a directive you register as `se` keeps its meaning.
- Changing the syntax recompiles the views (it is part of the compiled-view cache key).
