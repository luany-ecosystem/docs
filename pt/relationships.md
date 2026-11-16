# Relações

As relações são métodos públicos de um modelo que devolvem um objeto de relação. Aceda-lhes como propriedades para as carregar de forma preguiçosa (lazy loading), ou carregue-as antecipadamente com `with()` (eager loading) para evitar o problema das N+1 queries.

```php
use Luany\Database\Model;
use Luany\Database\Relations\HasMany;

class User extends Model
{
    protected string $table = 'users';

    public function posts(): HasMany
    {
        return $this->hasMany(Post::class);
    }
}

$user->posts;   // array<Post> — consultado uma vez e depois guardado em cache na instância
```

> **Nota** Os métodos de relação têm de ser `public`, não podem ter argumentos obrigatórios e devem declarar como tipo de retorno uma relação (ou nenhum tipo). Os métodos auxiliares (`hasMany()`, `belongsTo()`, …) são `protected` em `Model`.

Os objetos de relação são descritores, não query builders: `$user->posts()->where(…)` não existe. Filtre com [eager loading com restrições](#restringir-o-eager-loading) ou com uma query de modelo (`Post::query()->where('user_id', '=', $user->id)`).

## Convenções de chaves

As chaves estrangeiras são deduzidas a partir dos **nomes das tabelas**, no singular: tabela `users` → `user_id`, `categories` → `category_id`, `statuses` → `status_id`, `people` → `person_id`. Para casos fora do comum, indique as chaves explicitamente.

## Um para um

```php
use Luany\Database\Relations\HasOne;
use Luany\Database\Relations\BelongsTo;

class User extends Model
{
    public function profile(): HasOne
    {
        return $this->hasOne(Profile::class);   // profiles.user_id → users.id
    }
}

class Profile extends Model
{
    public function user(): BelongsTo
    {
        return $this->belongsTo(User::class);   // profiles.user_id → users.id
    }
}

$user->profile;    // ?Profile
$profile->user;    // ?User
```

| Método | Assinatura | Predefinições |
| --- | --- | --- |
| `hasOne` | `hasOne(string $related, ?string $foreignKey = null, ?string $localKey = null)` | FK `{this_table_singular}_id` na tabela relacionada; chave local = chave primária |
| `belongsTo` | `belongsTo(string $related, ?string $foreignKey = null, ?string $ownerKey = null)` | FK `{related_table_singular}_id` nesta tabela; chave do dono = chave primária do modelo relacionado |

## Um para muitos

```php
class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class, 'author_id');   // FK explícita
    }
}

class Post extends Model
{
    public function author(): BelongsTo
    {
        return $this->belongsTo(User::class, 'author_id');
    }
}

foreach ($user->posts as $post) {
    echo $post->title;
}
```

`hasMany(string $related, ?string $foreignKey = null, ?string $localKey = null)` devolve um array (vazio quando não há linhas).

## Muitos para muitos

`belongsToMany` faz a junção através de uma tabela pivot. Por predefinição, o nome da pivot é formado pelos nomes das duas tabelas no singular, por ordem alfabética (`roles` + `users` → `role_user`):

```php
use Luany\Database\Relations\BelongsToMany;

class User extends Model
{
    public function roles(): BelongsToMany
    {
        return $this->belongsToMany(Role::class)->withPivot('granted_at');
    }
}

foreach ($user->roles as $role) {
    echo $role->name, ' since ', $role->pivot['granted_at'];
}
```

```php
belongsToMany(
    string $related,
    ?string $pivotTable = null,        // role_user
    ?string $foreignPivotKey = null,   // user_id
    ?string $relatedPivotKey = null,   // role_id
    ?string $parentKey = null,         // users.id
    ?string $relatedKey = null,        // roles.id
)
```

`$model->pivot` é um array com as duas chaves da pivot mais as colunas indicadas em `withPivot()`.

### Associar, desassociar e sincronizar

```php
$user->roles()->attach(3);                                  // um id
$user->roles()->attach([1, 2]);                             // vários
$user->roles()->attach(4, ['granted_at' => date('Y-m-d')]); // colunas extra da pivot
$user->roles()->attach([5 => ['granted_at' => '2026-11-11']]);

$user->roles()->detach(2);       // devolve o número de linhas removidas
$user->roles()->detach();        // remove todas

$changes = $user->roles()->sync([1, 3]);
// ['attached' => [3], 'detached' => [2]]

$user->roles()->sync([4], detaching: false);   // apenas acrescenta
```

`sync()` é executado numa transação. Os ids também podem ser instâncias de modelos. Estes métodos lançam `\LogicException` se o modelo-pai ainda não tiver sido guardado. Não atualizam um `$user->roles` já carregado — chame `$user->refresh()` para o recarregar.

## Has many through

Chegue a linhas distantes através de um modelo intermédio — países → utilizadores → posts:

```php
use Luany\Database\Relations\HasManyThrough;

class Country extends Model
{
    public function posts(): HasManyThrough
    {
        return $this->hasManyThrough(Post::class, User::class);
        // users.country_id → countries.id, posts.user_id → users.id
    }
}
```

```php
hasManyThrough(
    string $related,
    string $through,
    ?string $firstKey = null,        // FK na tabela intermédia (country_id)
    ?string $secondKey = null,       // FK na tabela relacionada (user_id)
    ?string $localKey = null,        // chave neste modelo (id)
    ?string $secondLocalKey = null,  // chave no modelo intermédio (id)
)
```

Tanto o lazy loading como o eager loading usam uma única query com JOIN.

## Relações polimórficas

Uma tabela pode pertencer a vários modelos. Uma tabela `comments` com `commentable_type` e `commentable_id` (`$table->morphs('commentable')` numa [migration](migrations.md)) pode guardar comentários de posts e de vídeos:

```php
use Luany\Database\Relations\MorphMany;
use Luany\Database\Relations\MorphOne;
use Luany\Database\Relations\MorphTo;

class Post extends Model
{
    public function comments(): MorphMany
    {
        return $this->morphMany(Comment::class, 'commentable');
    }

    public function image(): MorphOne
    {
        return $this->morphOne(Image::class, 'imageable');
    }
}

class Comment extends Model
{
    public function commentable(): MorphTo
    {
        return $this->morphTo();   // nome obtido a partir do método: commentable_type / commentable_id
    }
}

$post->comments;                                   // array<Comment>
$comment->commentable;                             // Post, Video … ou null
$post->comments()->create(['body' => 'Nice!']);    // preenche type/id, respeita $fillable
$post->comments()->save($comment);

$comment->commentable()->associate($video);        // preenche type/id …
$comment->save();                                  // … e depois guarda
```

| Método | Assinatura |
| --- | --- |
| `morphMany` / `morphOne` | `(string $related, string $name, ?string $type = null, ?string $id = null, ?string $localKey = null)` |
| `morphTo` | `(?string $name = null, ?string $type = null, ?string $id = null, ?string $ownerKey = null)` |

### Morph map

Por predefinição, a coluna `_type` guarda o nome da classe. Registe aliases curtos uma única vez (por exemplo, num service provider) para desacoplar os dados guardados dos nomes das classes:

```php
Model::morphMap([
    'post'  => Post::class,
    'video' => Video::class,
]);
```

Ler um tipo desconhecido lança `\UnexpectedValueException`.

## Eager loading

O lazy loading executa uma query por modelo. O eager loading executa uma query por relação para todos os modelos:

```php
$posts = Post::with('author')->get();                 // 2 queries, não N + 1
$users = User::with('posts', 'profile')->all();
$user  = User::with('roles')->find(1);
```

`Model::with()` devolve uma `ModelQuery`, por isso combina-se com qualquer método de query:

```php
$posts = Post::with('author')->where('published', '=', 1)->paginate(20, $page);
```

### Relações aninhadas

A notação de pontos carrega relações de relações, com uma query por nível:

```php
$users = User::with('posts.comments.author')->get();
```

O aninhamento também funciona através de `morphTo`: os filhos são agrupados por classe.

### Restringir o eager loading

Passe um array no formato relação => closure. A closure recebe uma `ModelQuery` do modelo relacionado:

```php
use Luany\Database\ModelQuery;

$users = User::with([
    'posts' => fn (ModelQuery $q) => $q->where('published', '=', 1)->orderByDesc('published_at'),
    'roles',
])->get();

// A restrição aplica-se ao último segmento
User::with(['posts.comments' => fn (ModelQuery $q) => $q->where('approved', '=', 1)])->get();
```

> **Nota** Um `limit()` numa restrição limita toda a query agregada, não cada modelo-pai individualmente.

## Consultar a existência de relações

Filtre os modelos-pai pelas suas linhas relacionadas (uma subquery `EXISTS` / `COUNT` correlacionada, por isso as relações de uma tabela consigo própria funcionam):

```php
// Utilizadores com pelo menos um post
User::query()->has('posts')->get();

// Utilizadores com pelo menos 3 posts
User::query()->has('posts', '>=', 3)->get();

// Utilizadores com um post publicado com 100 ou mais visualizações
User::query()->whereHas('posts', fn (ModelQuery $q) => $q
    ->where('published', '=', 1)
    ->where('views', '>=', 100))->get();

// Aninhado
User::query()->whereHas('posts.comments', fn (ModelQuery $q) => $q->where('flagged', '=', 1))->get();

// Sem
User::query()->doesntHave('posts')->get();
User::query()->whereDoesntHave('roles', fn (ModelQuery $q) => $q->where('name', '=', 'admin'))->get();

// OU
User::query()->where('vip', '=', 1)->orWhereHas('orders')->get();
```

| Método | Assinatura |
| --- | --- |
| `whereHas` | `whereHas(string $relation, ?\Closure $callback = null, string $operator = '>=', int $count = 1)` |
| `orWhereHas` | igual, combinado com OR |
| `whereDoesntHave` | `whereDoesntHave(string $relation, ?\Closure $callback = null)` |
| `has` | `has(string $relation, string $operator = '>=', int $count = 1)` |
| `doesntHave` | `doesntHave(string $relation)` |

Operadores de contagem: `=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`. As linhas relacionadas eliminadas com soft delete são excluídas. As relações `morphTo` não suportam queries de existência (`\LogicException`).

## Contar linhas relacionadas

`withCount()` acrescenta um atributo `{relation}_count` sem carregar as linhas:

```php
$users = User::query()
    ->withCount('posts', [
        'posts as published_posts' => fn (ModelQuery $q) => $q->where('published', '=', 1),
    ])
    ->orderByDesc('posts_count')
    ->get();

echo $users[0]->posts_count, ' / ', $users[0]->published_posts;
```

Os nomes de relações em camelCase passam a snake_case (`blogPosts` → `blog_posts_count`). Os aliases têm de ser identificadores válidos.
