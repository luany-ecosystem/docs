# Controllers

Os controllers agrupam numa classe o tratamento de pedidos relacionados. Os seus métodos são ações de rota: recebem o pedido, os parâmetros da rota e quaisquer serviços de que precisem, e devolvem uma resposta.

## Criar um controller

```bash
luany make:controller Post      # app/Controllers/PostController.php
```

Os controllers ficam em `app/Controllers`, sob o namespace `App\Controllers`, e estendem a classe base `App\Controllers\Controller`:

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

Registe os métodos como ações de rota:

```php
Route::get('/posts', [PostController::class, 'index']);
Route::post('/posts', [PostController::class, 'store']);
```

Um nome de controller sem namespace (`['PostController', 'index']`) é procurado primeiro em `App\Controllers\` e depois em `App\Http\Controllers\`. Prefira `PostController::class` com uma instrução `use`, para que o editor e a análise estática consigam seguir a referência.

> **Nota** Mantenha o middleware fora dos controllers. Associe-o à rota ou ao Kernel — consulte [Middleware](middleware.md).

## Valores devolvidos

| Devolve | Resposta |
| --- | --- |
| `Response` (`redirect()`, `Response::json()`, …) | Enviada tal como está |
| `string` (p. ex. `view(...)`) | HTML, status 200 |
| `array` ou objeto | JSON |

Consulte [Pedidos e respostas](requests.md) para a API completa de `Response`.

## Injeção no construtor

Os controllers são construídos através do [container de serviços](container.md). Declare o tipo das dependências no construtor e elas são injetadas:

```php
use Psr\Log\LoggerInterface;

class ReportController extends Controller
{
    public function __construct(private LoggerInterface $logger) {}
}
```

## Injeção nos métodos

Os argumentos das ações também são resolvidos através do container. Cada parâmetro é preenchido segundo a primeira regra que se aplicar:

| Ordem | Regra | Exemplo |
| --- | --- | --- |
| 1 | **Por nome** — um parâmetro de rota com o mesmo nome | `string $slug` para `/posts/{slug}` |
| 2 | **Por tipo** — o `Request` atual, ou um serviço do container | `Request $request`, `Mailer $mailer` |
| 3 | **Por posição** — os valores restantes (primeiro o pedido, depois os parâmetros da rota), por ordem | `$request, $id` |
| 4 | **Predefinição** — o valor predefinido do parâmetro e, depois, `null` se for anulável | `int $page = 1` |

Assim, a ordem dos parâmetros não importa quando os nomes coincidem:

```php
Route::get('/users/{user}/posts/{post}', [UserPostController::class, 'show']);
```

```php
public function show(string $post, Request $request, PostRepository $posts, string $user): string
{
    // $post e $user por nome, $request e $posts por tipo
}
```

As assinaturas posicionais mais antigas continuam a funcionar: `show($request, $id)` continua a receber o pedido e depois o parâmetro.

> **Dica** Dê aos parâmetros das ações os mesmos nomes dos placeholders da rota. Um parâmetro sem correspondência e sem tipo é preenchido por posição, e o primeiro valor posicional é o pedido.

### Tipos escalares

Os parâmetros de rota chegam como strings. Quando a ação declara `int` ou `float` (incluindo anuláveis), o router converte o segmento antes de a chamar; um segmento que não seja um número válido para esse tipo responde com **404 Not Found**, antes de o middleware de rota ser executado:

```php
Route::get('/posts/{id}', [PostController::class, 'show']);
```

```php
public function show(int $id): string   // /posts/42 → 42; /posts/abc e /posts/4.2 → 404
{
    // ...
}
```

`int` aceita dígitos com sinal opcional, dentro do intervalo de inteiros do PHP; `float` aceita qualquer string numérica sem espaços à volta. Os outros tipos — `string`, sem tipo, ou uma união com um tipo não numérico — recebem a string original. Os parâmetros com [model binding](#model-binding) ficam a cargo do binding.

### Model binding

Com o [model binding nas rotas](routing.md#model-binding-nas-rotas), declare o modelo como tipo:

```php
Route::model('post', Post::class);
Route::get('/posts/{post}', [PostController::class, 'show']);
```

```php
public function show(Post $post): string    // 404 automático quando não é encontrado
public function show(?Post $post): string   // recebe null quando não é encontrado
```

Quando o binding devolve `null` e o parâmetro é de um tipo de objeto não anulável, o Kernel responde com **404 Not Found**.

## Closures

As ações com closures beneficiam da mesma resolução de argumentos:

```php
Route::get('/hello/{name}', fn (string $name) => "Hello, {$name}");
```

As closures não podem ser guardadas na [cache de rotas](routing.md#cache-de-rotas); prefira controllers para tudo o que vá além de rotas pequenas.

## Controllers de recurso

Um controller usado com `Route::resource()` implementa os métodos convencionais:

| Método | Rota |
| --- | --- |
| `index()` | `GET /posts` |
| `create()` | `GET /posts/create` |
| `store(Request $request)` | `POST /posts` |
| `show(string $id)` | `GET /posts/{id}` |
| `edit(string $id)` | `GET /posts/{id}/edit` |
| `update(Request $request, string $id)` | `PUT` / `PATCH /posts/{id}` |
| `destroy(string $id)` | `DELETE /posts/{id}` |

`luany make:feature Post title:string body:text` gera um controller de recurso completo, com o respetivo modelo, migration, vistas e rotas — consulte [make:feature](make-feature.md).
