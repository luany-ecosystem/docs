# Paginação

`paginate()` divide uma query em páginas. Executa duas queries — um `COUNT` e um `SELECT … LIMIT … OFFSET` — e devolve um `Luany\Database\PaginationResult` imutável com as linhas e os metadados necessários para renderizar a navegação.

## Paginar queries

Tanto o [query builder](query-builder.md) como as [queries de modelo](models.md#queries-de-modelo) o suportam:

```php
paginate(int $perPage = 15, int $page = 1): PaginationResult
```

O número da página **não** é lido automaticamente do pedido — tem de o passar:

```php
use App\Models\Post;
use Luany\Core\Http\Request;

class PostController extends Controller
{
    public function index(Request $request): string
    {
        $page  = max(1, (int) $request->query('page', 1));

        $posts = Post::query()
            ->where('published', '=', 1)
            ->orderByDesc('published_at')
            ->with('author')
            ->paginate(10, $page);

        return view('pages.posts.index', ['posts' => $posts]);
    }
}
```

Com uma query de modelo, `$posts->data` contém modelos (com o eager loading aplicado); com o query builder, contém arrays:

```php
$page = (new QueryBuilder(app('connection')))
    ->table('users')
    ->where('active', '=', 1)
    ->orderBy('name')
    ->paginate(perPage: 25, page: 2);
```

Um `$perPage` inferior a 1 lança `\InvalidArgumentException`; um `$page` inferior a 1 é tratado como 1. Uma página para lá da última devolve `data` vazio. A contagem respeita joins, `DISTINCT` e `GROUP BY`.

> **Dica** Acrescente sempre um `orderBy()`: sem ele, a base de dados pode devolver as linhas por uma ordem diferente em cada página.

## O `PaginationResult`

| Membro | Tipo | Descrição |
| --- | --- | --- |
| `$data` | `array` | Linhas (arrays) ou modelos desta página. |
| `$total` | `int` | Linhas correspondentes em todas as páginas. |
| `$perPage` | `int` | Tamanho da página. |
| `$currentPage` | `int` | Página atual (a contar de 1). |
| `$lastPage` | `int` | Última página; no mínimo `1`. |
| `$from` / `$to` | `?int` | Posição (a contar de 1) da primeira / última linha desta página; `null` quando vazia. |
| `hasMore(): bool` | | Existe uma página seguinte. |
| `hasPrev(): bool` | | Existe uma página anterior. |
| `toArray(): array` | | `data`, `total`, `per_page`, `current_page`, `last_page`, `from`, `to`, `has_more`, `has_prev`. |

Todas as propriedades são `readonly`.

## Renderizar links em LTE

O `PaginationResult` não gera HTML; construa os links numa vista. Um partial reutilizável, `views/components/pagination.lte`:

```lte
@if($paginator->lastPage > 1)
<nav class="pagination" aria-label="Pagination">
    @if($paginator->hasPrev())
        <a href="?page={{ $paginator->currentPage - 1 }}" rel="prev">&laquo; Previous</a>
    @endif

    @for($p = 1; $p <= $paginator->lastPage; $p++)
        @if($p === $paginator->currentPage)
            <span aria-current="page">{{ $p }}</span>
        @else
            <a href="?page={{ $p }}">{{ $p }}</a>
        @endif
    @endfor

    @if($paginator->hasMore())
        <a href="?page={{ $paginator->currentPage + 1 }}" rel="next">Next &raquo;</a>
    @endif
</nav>
@endif
```

Utilize-o a partir de uma página:

```lte
@extends('layouts.main')

@section('content')
    <p>Showing {{ $posts->from ?? 0 }}–{{ $posts->to ?? 0 }} of {{ $posts->total }} posts</p>

    @forelse($posts->data as $post)
        <article>
            <h2>{{ $post->title }}</h2>
            <small>by {{ $post->author?->name }}</small>
        </article>
    @empty
        <p>No posts yet.</p>
    @endforelse

    @include('components.pagination', ['paginator' => $posts])
@endsection
```

Para manter outros parâmetros da query string (filtros, termos de pesquisa), construa cada link com `http_build_query()`:

```lte
<a href="?{{ http_build_query(array_merge($filters, ['page' => $p])) }}">{{ $p }}</a>
```

Com muitas páginas, renderize uma janela em torno da página atual em vez de todos os números, por exemplo `@for($p = max(1, $paginator->currentPage - 2); $p <= min($paginator->lastPage, $paginator->currentPage + 2); $p++)`.

## APIs JSON

`toArray()` serializa a página e os seus metadados. Com queries de modelo, converta primeiro os modelos:

```php
$posts = Post::query()->orderByDesc('id')->paginate(20, $page);

return Response::json([
    ...$posts->toArray(),
    'data' => array_map(fn (Post $post) => $post->toArray(), $posts->data),
]);
```
