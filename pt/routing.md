# Rotas

As rotas associam um URI e um método HTTP a uma ação. São registadas com a facade `Luany\Core\Routing\Route` nos ficheiros da pasta `routes/`.

## Ficheiros de rotas

O Kernel carrega primeiro o `routes/http.php` e depois todos os outros ficheiros `*.php` em `routes/`, por ordem alfabética. Divida as rotas por área sempre que um ficheiro crescer demasiado:

```
routes/
  http.php      ← carregado primeiro
  admin.php
  api.php
  posts.php     ← gerado por: luany make:feature Post
```

Execute `luany route:list` para ver todas as rotas registadas.

## Rotas básicas

```php
use App\Controllers\PostController;
use Luany\Core\Http\Request;
use Luany\Core\Routing\Route;

Route::get('/', fn () => 'Hello, Luany!');

Route::get('/posts', [PostController::class, 'index']);
Route::post('/posts', [PostController::class, 'store']);
Route::put('/posts/{id}', [PostController::class, 'update']);
Route::patch('/posts/{id}', [PostController::class, 'update']);
Route::delete('/posts/{id}', [PostController::class, 'destroy']);

Route::any('/ping', fn () => 'pong');   // todos os métodos
```

Uma ação é uma closure ou um par `[Controller::class, 'method']`. Consulte [Controllers](controllers.md).

Os pedidos `HEAD` são respondidos pela rota `GET` correspondente, sem corpo. As rotas são avaliadas pela ordem de registo; a primeira que corresponder é a escolhida.

Os URIs podem conter pontos, como em `Route::get('/sitemap.xml', ...)` ou `/feed.json`. Correspondem da mesma forma com `luany serve` / `luany dev` e atrás do Apache ou do Nginx.

### Valores devolvidos

| A ação devolve | Resposta |
| --- | --- |
| `Response` | Enviada tal como está |
| `string` | Corpo HTML, status 200 |
| `array` ou objeto | JSON (`Response::json()`) |
| `null` | Corpo vazio, status 200 |

## Parâmetros de rota

Coloque um segmento entre chavetas para o capturar. Um parâmetro corresponde a um segmento do caminho (qualquer coisa exceto `/`):

```php
Route::get('/posts/{id}', function (Request $request, string $id) {
    return "Post {$id}";
});

Route::get('/users/{user}/posts/{post}', [UserPostController::class, 'show']);
```

Os parâmetros são passados à ação **pelo nome**, por isso a ordem em que aparecem na assinatura não importa. Os nomes dos parâmetros têm de começar por uma letra ou um underscore. Parâmetros opcionais e restrições com expressões regulares não são suportados — valide o valor na ação e use `abort(404)` quando for inválido.

### Parâmetros numéricos

Um parâmetro cujo argumento na ação é declarado `int` ou `float` é convertido para esse tipo. Um segmento que não seja um número válido responde com **404**, antes de o middleware de rota ser executado:

```php
Route::get('/posts/{id}', fn (int $id) => "Post #{$id}");   // /posts/7 → 7, /posts/abc → 404
```

Consulte [Controllers](controllers.md#tipos-escalares) para as regras exatas.

### Method spoofing em formulários

Os formulários HTML só conseguem enviar `GET` e `POST`. Adicione um campo `_method` a um formulário `POST` para chegar a uma rota `PUT`, `PATCH` ou `DELETE`:

```lte
<form method="POST" action="/posts/{{ $post->id }}">
    @csrf
    @method('DELETE')
    <button>Delete</button>
</form>
```

## Rotas com nome

Dê um nome a uma rota com `name()` (ou com o terceiro argumento) e gere o respetivo URL com o helper `route()`:

```php
Route::get('/posts/{id}', [PostController::class, 'show'])->name('posts.show');
Route::get('/about', [PageController::class, 'about'], 'about');
```

```php
route('posts.show', ['id' => 5]);            // http://localhost:8000/posts/5
route('posts.show', ['id' => 5, 'tab' => 'comments']);
                                             // http://localhost:8000/posts/5?tab=comments
```

Os valores são codificados para URL e prefixados com `APP_URL`; os parâmetros que não constam do padrão passam para a query string. Um parâmetro em falta lança uma `InvalidArgumentException`; um nome desconhecido lança uma `RuntimeException`.

## Grupos de rotas

Os grupos partilham um prefixo de URI e/ou middleware:

```php
use App\Http\Middleware\AuthMiddleware;

Route::prefix('admin')->middleware(AuthMiddleware::class)->group(function () {
    Route::get('/dashboard', [AdminController::class, 'dashboard']);   // /admin/dashboard
    Route::get('/users', [AdminController::class, 'users']);           // /admin/users
});

Route::middleware([AuthMiddleware::class, AuditMiddleware::class])->group(function () {
    Route::get('/account', [AccountController::class, 'show']);
});

// Forma com array
Route::group(['prefix' => '/api', 'middleware' => [ApiMiddleware::class]], function () {
    Route::get('/status', fn () => ['ok' => true]);
});
```

Os grupos podem ser aninhados: os prefixos são concatenados e as listas de middleware são combinadas, começando pela mais exterior.

> **Nota** Chamar `middleware()` duas vezes no mesmo grupo substitui a primeira lista. Passe todo o middleware de um grupo num único array.

As opções de grupo cobrem apenas `prefix` e `middleware` — os nomes das rotas não recebem prefixo, por isso dê a cada rota o nome completo.

## Rotas de recurso

`Route::resource()` regista as sete rotas CRUD convencionais para um controller:

```php
Route::resource('posts', PostController::class);
```

| Método | URI | Ação | Nome |
| --- | --- | --- | --- |
| GET | `/posts` | `index` | `posts.index` |
| GET | `/posts/create` | `create` | `posts.create` |
| POST | `/posts` | `store` | `posts.store` |
| GET | `/posts/{id}` | `show` | `posts.show` |
| GET | `/posts/{id}/edit` | `edit` | `posts.edit` |
| PUT | `/posts/{id}` | `update` | `posts.update` |
| PATCH | `/posts/{id}` | `update` | — |
| DELETE | `/posts/{id}` | `destroy` | `posts.destroy` |

Limite as ações com `only` ou `except`; o `apiResource()` deixa de fora as duas rotas de formulário (`create`, `edit`):

```php
Route::resource('photos', PhotoController::class, ['only' => ['index', 'show']]);
Route::resource('tags', TagController::class, ['except' => ['destroy']]);
Route::apiResource('comments', CommentController::class);
```

As rotas de recurso usam sempre `{id}` como nome do parâmetro.

## Rotas de vista

Quando uma rota apenas apresenta um template, dispense o controller:

```php
Route::view('/about', 'pages.about');
Route::view('/terms', 'pages.legal', ['title' => 'Terms of Service'], 'terms');
```

A vista recebe exatamente os dados que lhe passar. Os dados do pedido nunca são incluídos.

## Model binding nas rotas

O model binding substitui o valor bruto de um parâmetro por um objeto antes de a ação ser executada.

`Route::model()` resolve um parâmetro com o método `find()` do modelo:

```php
use App\Models\Post;

Route::model('post', Post::class);

Route::get('/posts/{post}', [PostController::class, 'show']);
```

```php
public function show(Post $post): string
{
    return view('pages.posts.show', compact('post'));
}
```

`Route::bind()` regista qualquer resolver — por exemplo, uma pesquisa por slug:

```php
Route::bind('post', fn (string $slug) => Post::firstWhere('slug = ?', [$slug]));
```

Um binding aplica-se a **todas** as rotas que tenham um parâmetro com esse nome. Quando o resolver devolve `null`:

| Parâmetro da ação | Resultado |
| --- | --- |
| `Post $post` (não anulável) | 404 Not Found |
| `?Post $post` | A ação recebe `null` |

## Erros: 404 e 405

Um URI que não corresponde a nenhuma rota produz um **404**. Um URI que corresponde a uma rota registada para outro método produz um **405 Method Not Allowed**, com um cabeçalho `Allow` que lista os métodos aceites. Ambos são apresentados pelo [handler de erros](errors.md).

## Cache de rotas

O router pode exportar a sua tabela de rotas para um ficheiro PHP e voltar a carregá-la sem executar os ficheiros de rotas:

```php
$path = base_path('storage/cache/routes.php');

Route::cache($path);        // escreve a tabela atual (escrita atómica)
Route::loadCache($path);    // true quando o ficheiro foi carregado
Route::clearCache($path);   // apaga o ficheiro
```

Uma tabela de rotas só pode ir para cache quando todas as ações e todo o middleware sobrevivem a `var_export()`:

| Pode ir para cache | Não pode ir para cache |
| --- | --- |
| Ações `[Controller::class, 'method']` | Ações com closures |
| Rotas `Route::view()` | Middleware passado como objetos sem `__set_state()` |
| Middleware passado como nome de classe | Resolvers de `Route::bind()` (closures) |
| Bindings de `Route::model()` | |

O `Route::cache()` lança uma `RuntimeException` com a lista de todas as rotas problemáticas, em vez de as descartar silenciosamente.

### Usar a cache no Kernel

Quando `storage/cache/routes.php` existe, o Kernel HTTP carrega as rotas a partir dele e **não executa os ficheiros de rotas**. Gere-o depois de carregar os ficheiros de rotas — por exemplo, com um pequeno script executado em cada deploy:

```php
<?php
// bin/route-cache.php
use Luany\Core\Routing\Route;
use Luany\Framework\Http\Kernel;

$app    = require __DIR__ . '/../bootstrap/app.php';
$kernel = $app->make(Kernel::class);
$path   = $kernel->routeCachePath();       // base_path('storage/cache/routes.php')

Route::clearCache($path);                  // para que boot() leia routes/*.php e não uma cache antiga
$kernel->boot();
Route::cache($path);

echo "Routes cached: {$path}\n";
```

```bash
php bin/route-cache.php
```

- Um ficheiro de cache é usado até ser apagado: volte a gerá-lo em cada deploy e execute `Route::clearCache()` (ou apague o ficheiro) sempre que as rotas mudarem. Não o crie em desenvolvimento.
- Os resolvers de `Route::bind()` são closures e não vão para a cache. Quando usar a cache, registe-os no método `boot()` de um service provider em vez de num ficheiro de rotas.
- Para mudar a localização, redefina `protected string $routeCacheFile` em `App\Http\Kernel` (relativo à raiz do projeto); defina-o como `''` para desativar a cache.

A CLI não tem um comando `route:cache`. Consulte [Deploy](deployment.md#rotas).
