# Pagination

`paginate()` splits a query into pages. It runs two queries — a `COUNT` and a `SELECT … LIMIT … OFFSET` — and returns an immutable `Luany\Database\PaginationResult` with the rows and the metadata to render navigation.

## Paginating queries

Both the [query builder](query-builder.md) and [model queries](models.md#model-queries) support it:

```php
paginate(int $perPage = 15, int $page = 1): PaginationResult
```

The page number is **not** read from the request automatically — pass it in:

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

With a model query, `$posts->data` holds models (eager loads applied); with the query builder it holds arrays:

```php
$page = (new QueryBuilder(app('connection')))
    ->table('users')
    ->where('active', '=', 1)
    ->orderBy('name')
    ->paginate(perPage: 25, page: 2);
```

A `$perPage` below 1 throws `\InvalidArgumentException`; a `$page` below 1 is treated as 1. A page beyond the last returns empty `data`. Counting honours joins, `DISTINCT` and `GROUP BY`.

> **Tip** Always add an `orderBy()`: without it, the database may return rows in a different order on each page.

## The `PaginationResult`

| Member | Type | Description |
| --- | --- | --- |
| `$data` | `array` | Rows (arrays) or models for this page. |
| `$total` | `int` | Matching rows across all pages. |
| `$perPage` | `int` | Page size. |
| `$currentPage` | `int` | Current page (1-based). |
| `$lastPage` | `int` | Last page; at least `1`. |
| `$from` / `$to` | `?int` | 1-based position of the first / last row on this page; `null` when empty. |
| `hasMore(): bool` | | A next page exists. |
| `hasPrev(): bool` | | A previous page exists. |
| `toArray(): array` | | `data`, `total`, `per_page`, `current_page`, `last_page`, `from`, `to`, `has_more`, `has_prev`. |

All properties are `readonly`.

## Rendering links in LTE

`PaginationResult` does not render HTML; build the links in a view. A reusable partial, `views/components/pagination.lte`:

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

Use it from a page:

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

To keep other query parameters (filters, search terms), build each link with `http_build_query()`:

```lte
<a href="?{{ http_build_query(array_merge($filters, ['page' => $p])) }}">{{ $p }}</a>
```

For many pages, render a window around the current page instead of every number, e.g. `@for($p = max(1, $paginator->currentPage - 2); $p <= min($paginator->lastPage, $paginator->currentPage + 2); $p++)`.

## JSON APIs

`toArray()` serialises the page and its metadata. With model queries, convert the models first:

```php
$posts = Post::query()->orderByDesc('id')->paginate(20, $page);

return Response::json([
    ...$posts->toArray(),
    'data' => array_map(fn (Post $post) => $post->toArray(), $posts->data),
]);
```
