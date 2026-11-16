# Sintaxe dos templates

O LTE (Luany Template Engine) compila ficheiros `.lte` para PHP simples e guarda o resultado em cache. Esta página explica como renderizar vistas, mostrar dados, escapar output, escrever comentários, usar estruturas de controlo e embutir PHP.

## Renderizar vistas

As vistas ficam em `views/` e usam a extensão `.lte`. Os nomes usam notação de pontos: `pages.home` corresponde a `views/pages/home.lte` (um ficheiro `.php` com o mesmo nome é aceite como alternativa).

Renderize uma vista com o helper `view()` e passe os dados num array; cada chave torna-se uma variável:

```php
use Luany\Core\Http\Request;

class HomeController extends Controller
{
    public function index(Request $request): string
    {
        return view('pages.home', ['title' => 'Welcome', 'user' => $user]);
    }
}
```

O framework regista o motor como `app('view')`, com as vistas em `views/`, os ficheiros compilados em `storage/cache/views` e recompilação automática quando `APP_DEBUG=true`. Fora do framework, crie o motor manualmente:

```php
use Luany\Lte\Engine;

$engine = new Engine(
    viewsPath:  __DIR__ . '/views',
    cachePath:  __DIR__ . '/storage/cache/views',
    autoReload: true,   // recompila em cada renderização (desenvolvimento)
);

echo $engine->render('pages.home', ['title' => 'Welcome']);
```

| Método do `Engine` | Descrição |
| --- | --- |
| `render(string $view, array $data = []): string` | Compila (se necessário), avalia e devolve o HTML. |
| `getCompiler(): Compiler` | Acede ao compilador, por exemplo para [registar diretivas](lte-custom.md). |
| `parseView(string $view): Ast\Template` | Analisa e valida sem renderizar (ver [Erros de template](lte-errors.md#inspecionar-a-árvore-sintática)). |
| `findView(string $view): ?string` | Caminho absoluto de uma vista, ou `null`. |
| `clearCache(): void` | Apaga todos os ficheiros compilados. |
| `setAutoReload(bool $autoReload): void` | Ativa ou desativa a recompilação em cada renderização. |
| `getCachePath(): string` | A pasta dos templates compilados. |

## Mostrar dados

Envolva uma expressão em `{{ }}` para a imprimir com escape de HTML:

```lte
<h1>{{ $title }}</h1>
<p>{{ $user->email }}</p>
<p>{{ strtoupper($user->name) }} has {{ count($posts) }} posts.</p>
```

É permitida qualquer expressão PHP. O output passa por `Luany\Lte\Escaper::escape()`, que:

- usa `htmlspecialchars()` com `ENT_QUOTES | ENT_SUBSTITUTE` e UTF-8, pelo que as aspas são codificadas e o UTF-8 inválido passa a U+FFFD em vez de uma string vazia;
- imprime `null` como string vazia;
- imprime um valor `Htmlable` tal como está (ver abaixo).

A expressão é compilada como `($expression) ?? ''`, por isso uma variável não definida não imprime nada em vez de gerar um aviso.

### Output sem escape

`{!! !!}` imprime o valor sem escape:

```lte
<div class="post-body">{!! $post->html !!}</div>
```

> **Aviso** Nunca use `{!! !!}` com conteúdo fornecido pelo utilizador. É a clássica porta aberta para XSS.

### HTML de confiança: `Htmlable` e `HtmlString`

Um objeto que implemente `Luany\Lte\Htmlable` (um único método, `toHtml(): string`) nunca é escapado por `{{ }}`, por valores inline de `@section` nem por valores predefinidos de `@yield`. `Luany\Lte\HtmlString` envolve uma string em que confia:

```php
use Luany\Lte\HtmlString;

return view('pages.about', [
    'badge' => new HtmlString('<span class="badge">New</span>'),
]);
```

```lte
<h2>About {{ $badge }}</h2>
```

Implemente `Htmlable` nos seus próprios value objects (um renderizador de Markdown, um widget de paginação) para que possam renderizar o seu markup através de `{{ }}`.

Os [slots](lte-layouts.md#componentes) dos componentes também são `HtmlString`, pelo que `{{ $slot }}` os imprime sem escape. `HtmlString::isEmpty()` / `isNotEmpty()` indicam se o HTML envolvido está vazio ou contém apenas espaços em branco.

## Comentários

Os comentários `{{-- --}}` são removidos em tempo de compilação e nunca chegam ao HTML:

```lte
{{-- Mostrado apenas aos administradores --}}
```

## Escapar a sintaxe LTE

| Escreve | Resultado | Caso de uso |
| --- | --- | --- |
| `@@if` | `@if` | Mostrar literalmente o nome de uma diretiva. |
| `@@` | `@` | Uma arroba isolada junto ao nome de uma diretiva. |
| `@{{ message }}` | `{{ message }}` | Frameworks do lado do cliente (Vue, Alpine). |

Os restantes sinais `@` não precisam de escape:

- **Palavras desconhecidas são texto.** Só as diretivas incorporadas e as [registadas](lte-custom.md) são interpretadas, por isso `@media`, `@keyframes` e `@username` ficam como estão.
- **`palavra@palavra` é texto.** Um `@` imediatamente a seguir a uma letra, dígito ou underscore não é uma diretiva, por isso `me@if.com` e `dev@php.net` são seguros. As diretivas de fecho (`@end…`, `@else`, `@elseif`, `@stop`) são a exceção — `Saved@endif` continua a fechar o bloco.

```lte
<style>@media (max-width: 600px) { .nav { display: none; } }</style>
<p>Contact support@luany.dev or type @{{ name }} in the template.</p>
```

## Estruturas de controlo

### Condicionais

```lte
@if($user->isAdmin())
    <a href="/admin">Dashboard</a>
@elseif($user->isEditor())
    <a href="/editor">Editor</a>
@else
    <a href="/profile">Profile</a>
@endif

@unless($user->verified)
    <p>Please verify your email.</p>
@endunless
```

`@isset`, `@ifempty` e `@empty($x)` testam uma variável; todas aceitam um ramo `@else`:

```lte
@isset($subtitle)
    <h2>{{ $subtitle }}</h2>
@endisset

@ifempty($cart)
    <p>Your cart is empty.</p>
@else
    <p>{{ count($cart) }} items</p>
@endifempty
```

`@empty($x) … @endempty` equivale a `@ifempty`. Sem argumentos, `@empty` é o ramo vazio de `@forelse`.

### Ciclos

```lte
@foreach($users as $user)
    <li>{{ $user->name }}</li>
@endforeach

@for($i = 1; $i <= 3; $i++)
    <span>Step {{ $i }}</span>
@endfor

@while($item = array_shift($queue))
    <p>{{ $item }}</p>
@endwhile
```

`@forelse` acrescenta um ramo para o caso vazio. Funciona com arrays, `Traversable`s e generators, e pode ser aninhado:

```lte
@forelse($posts as $post)
    <article>{{ $post->title }}</article>
@empty
    <p>No posts yet.</p>
@endforelse
```

`@empty` é opcional — `@forelse … @endforelse` sozinho comporta-se como `@foreach`.

> **Nota** O LTE não tem `@break`, `@continue` nem `@switch`. Use `@php(break)` / `@php(continue)` dentro de um ciclo, ou cadeias `@if` / `@elseif`. Pode registar as suas próprias com [diretivas próprias](lte-custom.md).

É permitido espaço entre a diretiva e os seus argumentos: `@if ($x)` funciona como `@if($x)`.

## PHP embutido

Forma inline, para uma única instrução:

```lte
@php($total = array_sum(array_column($items, 'price')))
<p>Total: {{ number_format($total, 2) }}</p>
```

Forma de bloco, fechada por `@endphp`. O conteúdo é copiado tal como está — a sintaxe LTE no seu interior não é interpretada:

```lte
@php
    $grouped = [];
    foreach ($items as $item) {
        $grouped[$item->category][] = $item;
    }
@endphp
```

> **Dica** Mantenha a lógica nos controllers. `@php` serve para pequenos valores locais da vista; tudo o que for maior pertence a classes PHP.

## Próximos passos

- [Layouts e componentes](lte-layouts.md) — `@extends`, `@section`, `@include`, `@component`.
- [Referência de diretivas](lte-directives.md) — todas as diretivas incorporadas.
- [Erros de template](lte-errors.md) — validação em tempo de compilação e cache.
