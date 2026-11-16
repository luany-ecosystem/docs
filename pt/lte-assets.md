# Assets e stacks

As vistas e os componentes precisam muitas vezes de um pouco de CSS ou JavaScript próprio. O LTE oferece duas ferramentas: **blocos de assets** (`@style`, `@script`), recolhidos de toda a renderização e impressos uma única vez, e **stacks** (`@push`, `@stack`), buffers nomeados a que pode acrescentar conteúdo a partir de qualquer vista.

## Estilos e scripts inline

Declare um bloco em qualquer lado — numa página, num partial, num componente:

```lte
{{-- views/components/card.lte --}}
<div class="card">{!! $slot !!}</div>

@style
    .card { padding: 1rem; border-radius: 8px; box-shadow: 0 1px 3px rgb(0 0 0 / .1); }
@endstyle

@script(defer)
    document.querySelectorAll('.card').forEach(card => card.classList.add('ready'));
@endscript
```

Depois, imprima no layout tudo o que foi recolhido:

```lte
<head>
    ...
    @styles
</head>
<body>
    ...
    @scripts
</body>
```

Resultado:

```html
<style>
.card { padding: 1rem; border-radius: 8px; box-shadow: 0 1px 3px rgb(0 0 0 / .1); }
</style>
...
<script defer>
document.querySelectorAll('.card').forEach(card => card.classList.add('ready'));
</script>
```

### Como funciona a recolha

- **Marcadores.** `@styles` e `@scripts` compilam para marcadores em forma de comentário HTML. O motor só os substitui depois de a página *e* todos os níveis de layout terem sido avaliados, por isso os blocos registados por partials incluídos mais tarde no layout (barra de navegação, rodapé, mensagens flash) continuam a aparecer — mesmo quando `@styles` está no `<head>`.
- **Sem duplicados.** Os blocos com conteúdo idêntico são mantidos uma única vez. Um componente renderizado cinquenta vezes contribui com o seu CSS uma só vez.
- **Ordem.** Os blocos são impressos pela ordem em que foram registados pela primeira vez.
- **Opções.** `@script(defer)` acrescenta `defer`. Não é suportado nenhum outro atributo.
- **Posicionamento.** Use `@styles` e `@scripts` no layout (ou na vista de topo). Cada um pode aparecer mais de uma vez; todas as ocorrências recebem o resultado completo.
- **Por renderização.** A recolha é reiniciada no início de cada `render()` de topo.

O conteúdo de `@style` / `@script` é copiado tal como está: `{{ }}` e as diretivas no interior continuam a funcionar, pelo que pode interpolar valores — mas lembre-se de que, como a eliminação de duplicados se baseia no conteúdo, cada valor diferente passa a contar como um bloco diferente.

> **Dica** Prefira ficheiros estáticos (`<link href="{{ asset('css/app.css') }}">`) para tudo o que é partilhado entre páginas. Os blocos de assets brilham em regras pequenas e locais a um componente.

## Stacks

Uma stack é um buffer nomeado. `@push` acrescenta-lhe conteúdo a partir de qualquer vista; `@stack` imprime o seu conteúdo:

```lte
{{-- views/pages/map.lte --}}
@extends('layouts.main')

@push('head')
    <link rel="stylesheet" href="https://unpkg.com/leaflet@1.9.4/dist/leaflet.css">
@endpush

@push('scripts')
    <script src="https://unpkg.com/leaflet@1.9.4/dist/leaflet.js"></script>
@endpush

@section('content')
    <div id="map"></div>
@endsection
```

```lte
{{-- views/layouts/main.lte --}}
<head>
    @stack('head')
</head>
<body>
    @yield('content')
    @stack('scripts')
</body>
```

Ao contrário das secções, os pushes nunca se substituem uns aos outros — cada `@push` acrescenta. As stacks **não** eliminam duplicados.

> **Nota** `@stack` imprime o que foi acrescentado *antes* de ser executado. Uma página é avaliada antes do seu layout, por isso os pushes da página chegam sempre às stacks do layout. Os pushes de um partial que o layout inclui *depois* da chamada a `@stack` perdem-se — coloque `@stack` abaixo desses includes, ou use `@style` / `@script`, cujos marcadores são resolvidos no fim.

## Qual usar

| | `@style` / `@script` | `@push` / `@stack` |
| --- | --- | --- |
| Conteúdo | Corpo de CSS / JS inline | Qualquer markup (`<link>`, `<meta>`, `<script src>`) |
| Tag envolvente | Acrescentada automaticamente | Escrita por si |
| Duplicados | Removidos | Mantidos |
| Resolução | Depois da renderização completa | Quando `@stack` é executado |
