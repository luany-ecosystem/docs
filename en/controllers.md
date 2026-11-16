# Controllers

Controllers group related request handling into a class. Their methods are route actions: they receive the request, route parameters and any services they ask for, and return a response.

## Creating a controller

```bash
luany make:controller Post      # app/Controllers/PostController.php
```

Controllers live in `app/Controllers` under the `App\Controllers` namespace and extend the base `App\Controllers\Controller` class:

```php
namespace App\Controllers;

use App\Models\Post;
use Luany\Core\Http\Request;
use Luany\Core\Http\Response;

class PostController extends Controller
{
    public function index(): string
    {
        return view('pages.posts.index', ['posts' => Post::all()]);
    }

    public function store(Request $request): Response
    {
        $data = validate($request->body(), [
            'title' => 'required|string|max:255',
            'body'  => 'required|string',
        ]);

        Post::create($data);
        flash('success', 'Post created.');

        return redirect('/posts');
    }
}
```

Register the methods as route actions:

```php
Route::get('/posts', [PostController::class, 'index']);
Route::post('/posts', [PostController::class, 'store']);
```

> **Note** Keep middleware out of controllers. Attach it to the route or the Kernel — see [Middleware](middleware.md).

## Return values

| Return | Response |
| --- | --- |
| `Response` (`redirect()`, `Response::json()`, …) | Sent as is |
| `string` (e.g. `view(...)`) | HTML, status 200 |
| `array` or object | JSON |

See [Requests & Responses](requests.md) for the full `Response` API.

## Constructor injection

Controllers are built through the [service container](container.md). Type-hint dependencies in the constructor and they are injected:

```php
use Psr\Log\LoggerInterface;

class ReportController extends Controller
{
    public function __construct(private LoggerInterface $logger) {}
}
```

## Method injection

Action arguments are resolved through the container too. Each parameter is filled using the first rule that applies:

| Order | Rule | Example |
| --- | --- | --- |
| 1 | **By name** — a route parameter with the same name | `string $slug` for `/posts/{slug}` |
| 2 | **By type** — the current `Request`, or a service from the container | `Request $request`, `Mailer $mailer` |
| 3 | **By position** — remaining values (the request first, then route parameters) in order | `$request, $id` |
| 4 | **Default** — the parameter's default value, then `null` if nullable | `int $page = 1` |

So the order of parameters does not matter when names match:

```php
Route::get('/users/{user}/posts/{post}', [UserPostController::class, 'show']);
```

```php
public function show(string $post, Request $request, PostRepository $posts, string $user): string
{
    // $post and $user by name, $request and $posts by type
}
```

Older positional signatures keep working: `show($request, $id)` still receives the request and then the parameter.

> **Tip** Name action parameters after the route placeholders. An unmatched, untyped parameter is filled by position, and the first positional value is the request.

### Scalar types

Route parameters arrive as strings. A scalar type such as `int $id` converts numeric strings, but a non-numeric segment (`/posts/abc`) raises a `TypeError` (a 500 error). Accept `string` and validate when the segment may be arbitrary:

```php
public function show(string $id): string
{
    if (!ctype_digit($id)) {
        abort(404);
    }
    // ...
}
```

### Model binding

With [route model binding](routing.md#route-model-binding), type-hint the model:

```php
Route::model('post', Post::class);
Route::get('/posts/{post}', [PostController::class, 'show']);
```

```php
public function show(Post $post): string    // 404 automatically when not found
public function show(?Post $post): string   // receives null when not found
```

When the binding returns `null` and the parameter is a non-nullable object type, the Kernel responds with **404 Not Found**.

## Closures

Closure actions get the same argument resolution:

```php
Route::get('/hello/{name}', fn (string $name) => "Hello, {$name}");
```

Closures cannot be stored in the [route cache](routing.md#route-caching); prefer controllers for anything beyond small routes.

## Resource controllers

A controller used with `Route::resource()` implements the conventional methods:

| Method | Route |
| --- | --- |
| `index()` | `GET /posts` |
| `create()` | `GET /posts/create` |
| `store(Request $request)` | `POST /posts` |
| `show(string $id)` | `GET /posts/{id}` |
| `edit(string $id)` | `GET /posts/{id}/edit` |
| `update(Request $request, string $id)` | `PUT` / `PATCH /posts/{id}` |
| `destroy(string $id)` | `DELETE /posts/{id}` |

`luany make:feature Post title:string body:text` generates a complete resource controller with its model, migration, views and routes — see [make:feature](make-feature.md).
