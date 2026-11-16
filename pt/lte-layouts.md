# Layouts e componentes

O LTE constrói páginas a partir de três peças: **layouts** que uma página estende, **partials** incluídos com `@include` e **componentes** com slots nomeados. Esta página cobre as três.

## Definir um layout

Um layout é uma vista normal que imprime secções com `@yield`. O skeleton inclui `views/layouts/main.lte`:

```lte
<!DOCTYPE html>
<html lang="{{ locale() }}">
<head>
    <meta charset="UTF-8">
    <title>@yield('title', config('app.name'))</title>
    <link rel="stylesheet" href="{{ asset('css/app.css') }}">
    @styles
    @stack('head')
</head>
<body>
    @include('components.flash')

    <main>
        @yield('content')
    </main>

    <script src="{{ asset('js/app.js') }}" defer></script>
    @scripts
    @stack('scripts')
</body>
</html>
```

`@yield('name', $default)` imprime uma secção, ou o valor predefinido quando a secção não está definida. O valor predefinido é escapado (envolva markup de confiança em [`HtmlString`](lte-syntax.md#html-de-confiança-htmlable-e-htmlstring)).

`@styles`, `@scripts` e `@stack` são explicados em [Assets e stacks](lte-assets.md).

## Estender um layout

Uma página declara o seu layout com `@extends` e preenche as secções:

```lte
@extends('layouts.main')

@section('title', 'Dashboard')

@section('content')
    <h1>Dashboard</h1>
    <p>Welcome back, {{ $user->name }}.</p>
@endsection
```

| Forma | Comportamento |
| --- | --- |
| `@section('name') … @endsection` | Captura o bloco. O conteúdo é HTML já renderizado e não volta a ser escapado. |
| `@section('name') … @stop` | Igual; `@stop` é um alias de `@endsection`. |
| `@section('name', $value)` | Valor inline, **com escape de HTML**. Passe um `HtmlString` para markup. |
| `@yield('name')` / `@yield('name', $default)` | Imprime a secção; o valor predefinido é escapado. |

A página é avaliada primeiro e só depois o layout. Numa página que estende um layout, o conteúdo fora de qualquer `@section` é descartado.

### Layouts multinível

Um layout pode estender outro layout. Quando a mesma secção é definida em vários níveis, prevalece a definição mais próxima da página:

```lte
{{-- views/layouts/admin.lte --}}
@extends('layouts.main')

@section('title', 'Admin')

@section('content')
    <div class="admin">
        <aside>@include('admin.sidebar')</aside>
        <section>@yield('admin-content')</section>
    </div>
@endsection
```

```lte
{{-- views/admin/users.lte --}}
@extends('layouts.admin')

@section('title', 'Users')   {{-- substitui "Admin" --}}

@section('admin-content')
    <h1>Users</h1>
@endsection
```

Uma cadeia circular (`a` estende `b`, que estende `a`) lança uma `TemplateSyntaxException`. Só a página renderizada e os seus layouts podem usar `@extends`; um `@extends` dentro de um partial incluído ou de um componente é ignorado, para que não possa sequestrar o layout da página.

Os nomes podem ser dinâmicos: `@extends($layout)`, `@section($name)`, `@yield($name)`.

## Incluir partials

`@include` renderiza outra vista no próprio local. O partial recebe todas as variáveis da vista-mãe; um segundo argumento junta dados adicionais:

```lte
@include('components.navbar')

@foreach($posts as $post)
    @include('posts.card', ['post' => $post, 'compact' => true])
@endforeach
```

As variáveis cujo nome começa por `__` são internas e não são passadas. O nome da vista pode ser qualquer expressão: `@include($widget)`, `@include('widgets.' . $type)`.

## Componentes

Os componentes são vistas reutilizáveis com slots. O conteúdo entre `@component` e `@endcomponent` passa a ser `$slot`; cada `@slot('name') … @endslot` torna-se uma variável com esse nome; o segundo argumento opcional de `@component` passa dados.

Ficheiro do componente `views/components/alert.lte`:

```lte
<div class="alert alert-{{ $type }}" role="alert">
    @isset($title)
        <strong>{{ $title }}</strong>
    @endisset
    {{ $slot }}
</div>
```

Utilização:

```lte
@component('components.alert', ['type' => 'error'])
    @slot('title')
        Something went wrong
    @endslot

    Please check the form and try again.
@endcomponent
```

| Variável no componente | Origem |
| --- | --- |
| `$slot` | Slot predefinido: tudo o que está fora dos slots nomeados, sem espaços nas pontas, como `HtmlString`. Está sempre definido (vazio quando quem chama não passa conteúdo). |
| `$title`, `$footer`, … | Slots nomeados, também `HtmlString`. Só estão definidos quando quem chama os fornece — proteja-os com `@isset`. |
| `$type`, … | Array de dados passado como segundo argumento de `@component`. |

Um slot nomeado sobrepõe-se a uma chave de dados com o mesmo nome, e `$slot` sobrepõe-se a ambos.

Os slots contêm HTML já renderizado, envolvido num `Luany\Lte\HtmlString` (ver [HTML de confiança](lte-syntax.md#html-de-confiança-htmlable-e-htmlstring)), pelo que `{{ $slot }}` os imprime tal como estão, em vez de os escapar uma segunda vez. `{!! $slot !!}` também funciona.

Um objeto é sempre verdadeiro, por isso `@if($slot)` é verdadeiro mesmo quando o slot está vazio. Teste o conteúdo com `isEmpty()` / `isNotEmpty()`, que consideram vazio o HTML vazio ou só com espaços em branco:

```lte
<div class="card">
    @isset($header)
        <header>{{ $header }}</header>
    @endisset
    @if($slot->isNotEmpty())
        <div class="card-body">{{ $slot }}</div>
    @endif
</div>
```

> **Nota** Alterado no LTE 1.3. Antes, os slots eram strings simples: `{{ $slot }}` escapava-os uma segunda vez e `@if($slot)` era falso para um slot vazio. Ao atualizar, substitua `@if($slot)` por `@if($slot->isNotEmpty())`; `{!! $slot !!}` não precisa de alterações.

Os componentes podem ser aninhados livremente, e um componente pode incluir partials ou outros componentes. Os blocos `@style` e `@script` dentro de um componente são recolhidos uma única vez, independentemente de quantas vezes o componente é renderizado (ver [Assets e stacks](lte-assets.md)).

> **Dica** Deixe um espaço ou uma quebra de linha entre uma diretiva de fecho e o texto seguinte: `@endslot Body`, e não `@endslotBody`. A forma colada é lida como texto; o LTE assinala-a em tempo de compilação com uma sugestão (ver [Erros de template](lte-errors.md#fechos-colados)).

## Qual usar

| Necessidade | Usar |
| --- | --- |
| Estrutura comum da página (head, navegação, rodapé) | Layout + `@extends` |
| Um fragmento que só precisa das variáveis atuais | `@include` |
| Um fragmento que envolve markup fornecido por quem chama | `@component` + `@slot` |
| Tags de `<head>` ou de script específicas de uma página | [`@push` / `@stack`](lte-assets.md#stacks) |
