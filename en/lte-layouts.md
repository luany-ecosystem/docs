# Layouts & Components

LTE builds pages from three pieces: **layouts** that a page extends, **partials** pulled in with `@include`, and **components** with named slots. This page covers all three.

## Defining a layout

A layout is an ordinary view that prints sections with `@yield`. The skeleton ships `views/layouts/main.lte`:

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

`@yield('name', $default)` prints a section, or the default when the section is not defined. The default is HTML-escaped (wrap trusted markup in [`HtmlString`](lte-syntax.md#trusted-html-htmlable-and-htmlstring)).

`@styles`, `@scripts` and `@stack` are covered in [Assets & Stacks](lte-assets.md).

## Extending a layout

A page declares its layout with `@extends` and fills sections:

```lte
@extends('layouts.main')

@section('title', 'Dashboard')

@section('content')
    <h1>Dashboard</h1>
    <p>Welcome back, {{ $user->name }}.</p>
@endsection
```

| Form | Behaviour |
| --- | --- |
| `@section('name') … @endsection` | Captures the block. Its content is rendered HTML and is not escaped again. |
| `@section('name') … @stop` | Same; `@stop` is an alias for `@endsection`. |
| `@section('name', $value)` | Inline value, **HTML-escaped**. Pass an `HtmlString` for markup. |
| `@yield('name')` / `@yield('name', $default)` | Prints the section; the default is escaped. |

The page is evaluated first, then the layout. Content outside any `@section` in a page that extends a layout is discarded.

### Multi-level layouts

A layout may extend another layout. When the same section is defined at several levels, the definition closest to the page wins:

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

@section('title', 'Users')   {{-- overrides "Admin" --}}

@section('admin-content')
    <h1>Users</h1>
@endsection
```

A circular chain (`a` extends `b` extends `a`) throws a `TemplateSyntaxException`. Only the rendered page and its layouts may use `@extends`; an `@extends` inside an included partial or component is ignored so it cannot hijack the page layout.

Names can be dynamic: `@extends($layout)`, `@section($name)`, `@yield($name)`.

## Including partials

`@include` renders another view in place. The partial receives every variable of the parent view; a second argument merges extra data:

```lte
@include('components.navbar')

@foreach($posts as $post)
    @include('posts.card', ['post' => $post, 'compact' => true])
@endforeach
```

Variables whose names start with `__` are internal and are not passed. The view name may be any expression: `@include($widget)`, `@include('widgets.' . $type)`.

## Components

Components are reusable views with slots. Content between `@component` and `@endcomponent` becomes `$slot`; each `@slot('name') … @endslot` becomes a variable with that name; the optional second argument of `@component` passes data.

Component file `views/components/alert.lte`:

```lte
<div class="alert alert-{{ $type }}" role="alert">
    @isset($title)
        <strong>{!! $title !!}</strong>
    @endisset
    {!! $slot !!}
</div>
```

Using it:

```lte
@component('components.alert', ['type' => 'error'])
    @slot('title')
        Something went wrong
    @endslot

    Please check the form and try again.
@endcomponent
```

| Variable in the component | Source |
| --- | --- |
| `$slot` | Default slot: everything outside named slots, trimmed. Always defined. |
| `$title`, `$footer`, … | Named slots. Defined only when the caller provides them — guard with `@isset`. |
| `$type`, … | Data array passed as the second argument of `@component`. |

A named slot overrides a data key with the same name, and `$slot` overrides both.

> **Warning** Slot content is already-rendered HTML in a plain string. Print it with `{!! $slot !!}`; `{{ $slot }}` would escape it a second time.

Components nest freely, and a component can include partials or other components. `@style` and `@script` blocks inside a component are collected once, however many times the component renders (see [Assets & Stacks](lte-assets.md)).

> **Tip** Put a space or newline between a closing directive and the following text: `@endslot Body`, not `@endslotBody`. The glued form is read as text; LTE reports it at compile time with a hint (see [Template Errors](lte-errors.md#glued-closers)).

## Choosing between them

| Need | Use |
| --- | --- |
| Shared page chrome (head, nav, footer) | Layout + `@extends` |
| A fragment that only needs the current variables | `@include` |
| A fragment wrapping caller-provided markup | `@component` + `@slot` |
| Page-specific `<head>` or script tags | [`@push` / `@stack`](lte-assets.md#stacks) |
