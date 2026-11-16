# Assets & Stacks

Views and components often need a bit of CSS or JavaScript of their own. LTE offers two tools: **asset blocks** (`@style`, `@script`), collected from the whole render and printed once, and **stacks** (`@push`, `@stack`), named buffers you append to from any view.

## Inline styles and scripts

Declare a block anywhere — a page, a partial, a component:

```lte
{{-- views/components/card.lte --}}
<div class="card">{{ $slot }}</div>

@style
    .card { padding: 1rem; border-radius: 8px; box-shadow: 0 1px 3px rgb(0 0 0 / .1); }
@endstyle

@script(module)
    document.querySelectorAll('.card').forEach(card => card.classList.add('ready'));
@endscript
```

Then print everything that was collected in the layout:

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

Output:

```html
<style>
.card { padding: 1rem; border-radius: 8px; box-shadow: 0 1px 3px rgb(0 0 0 / .1); }
</style>
...
<script type="module">
document.querySelectorAll('.card').forEach(card => card.classList.add('ready'));
</script>
```

### How collection works

- **Placeholders.** `@styles` and `@scripts` compile to HTML comment placeholders. The engine replaces them only after the page *and* every layout level have been evaluated, so blocks registered by partials included later in the layout (navbar, footer, flash messages) still appear — even when `@styles` sits in `<head>`.
- **Deduplication.** Blocks with identical content are kept once. A component rendered fifty times contributes its CSS once.
- **Order.** Blocks are printed in the order they were first registered.
- **Options.** `@script` accepts `module`, `defer` and `async`, separated by commas (see below).
- **Placement.** Use `@styles` and `@scripts` in the layout (or the top-level view). Each may appear more than once; every occurrence receives the full output.
- **Per render.** The collection is reset at the start of every top-level `render()`.

`@style` / `@script` content is copied as-is: `{{ }}` and directives inside still work, so you can interpolate values — but remember that content-based deduplication then sees each distinct value as a different block.

> **Tip** Prefer static files (`<link href="{{ asset('css/app.css') }}">`) for anything shared across pages. Asset blocks shine for small, component-local rules.

### Script options

| Option | Output | Effect |
| --- | --- | --- |
| `module` | `type="module"` | ES module: deferred by default, its own scope, `import` allowed. |
| `defer` | `defer` | Runs after the document has been parsed. |
| `async` | `async` | Runs as soon as it is ready. |

```lte
@script(module, async)
    import { init } from '/js/charts.js';
    init(document.querySelectorAll('[data-chart]'));
@endscript
```

```html
<script type="module" async>
import { init } from '/js/charts.js';
init(document.querySelectorAll('[data-chart]'));
</script>
```

> **Note** Browsers honour `defer` and `async` on inline scripts only together with `module`: a plain `@script(defer)` prints the attribute, but the script still runs immediately. Use `@script(module)` for deferred inline code. Unknown options are ignored. `module` and `async` need LTE 1.3; earlier versions accepted only `defer`.

## Stacks

A stack is a named buffer. `@push` appends to it from any view; `@stack` prints its contents:

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

Unlike sections, pushes never replace each other — every `@push` appends. Stacks are **not** deduplicated.

> **Note** `@stack` prints what has been pushed *before* it runs. A page is evaluated before its layout, so pushes from the page always reach the layout's stacks. Pushes from a partial that the layout includes *after* the `@stack` call are lost — place `@stack` below such includes, or use `@style` / `@script`, whose placeholders are resolved at the end.

## Choosing between them

| | `@style` / `@script` | `@push` / `@stack` |
| --- | --- | --- |
| Content | Inline CSS / JS bodies | Any markup (`<link>`, `<meta>`, `<script src>`) |
| Wrapping tag | Added for you | You write it |
| Duplicates | Removed | Kept |
| Resolved | After the full render | When `@stack` executes |
