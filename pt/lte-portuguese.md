# Diretivas em português

As vistas LTE podem ser escritas com as diretivas em português. `@se … @senao … @fimse` compila exatamente para o mesmo PHP que `@if … @else … @endif`. Os nomes ingleses continuam a funcionar, e os dois podem misturar-se no mesmo template.

Precisa do luany/lte 1.4 e do luany/framework 1.7.

## Ativar

A sintaxe é uma definição do projeto. Em `config/app.php`:

```php
'syntax' => env('LUANY_SYNTAX', 'en'),   // 'pt' acrescenta os nomes em português
```

e no `.env`:

```dotenv
LUANY_SYNTAX=pt
```

Fora de uma aplicação Luany, chame `$engine->setSyntax('pt')` no motor LTE.

Tem de ser ativada de propósito: no LTE, um `@palavra` que não é diretiva fica como texto. Com `pt`, palavras como `@se` ou `@para` passam a ser diretivas, o que poderia mudar uma página que as mostra como texto.

## Exemplo

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

## Referência

| Inglês | Português | Inglês | Português |
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

## Regras

- **Acentos e variantes ortográficas são aceites:** `@senão`, `@senãose`, `@método`; `@seccao`, `@seção` e `@secção` querem todos dizer `@secao`.
- **Só em minúsculas,** como as diretivas inglesas: `@Se` é texto.
- **Os nomes técnicos não têm forma portuguesa:** `@php`, `@csrf`, `@json`, `@style`, `@styles`, `@script`, `@scripts`, `@push`, `@stack`, `@slot`, `@dump`, `@dd`.
- **Os erros usam o nome que escreveu:** `Unexpected @fimse: @paracada opened on line 12 is still open (expected @fimparacada)`. O `luany view:check` e a extensão do VS Code seguem a sintaxe do projeto.
- **As [diretivas próprias](lte-custom.md) têm prioridade:** uma diretiva que registe como `se` mantém o seu significado.
- Mudar a sintaxe recompila as vistas (faz parte da chave da cache das vistas compiladas).
