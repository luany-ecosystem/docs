# Relationships

Relationships are public methods on a model that return a relation object. Access them as properties to load lazily, or eager-load them with `with()` to avoid N+1 queries.

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

$user->posts;   // array<Post> — queried once, then cached on the instance
```

> **Note** Relation methods must be `public`, take no required arguments and declare a relation return type (or none). The helper methods (`hasMany()`, `belongsTo()`, …) are `protected` on `Model`.

Relation objects are descriptors, not query builders: `$user->posts()->where(…)` does not exist. Filter with [constrained eager loading](#constraining-eager-loads) or a model query (`Post::query()->where('user_id', '=', $user->id)`).

## Key conventions

Foreign keys are guessed from **table names**, singularised: table `users` → `user_id`, `categories` → `category_id`, `statuses` → `status_id`, `people` → `person_id`. Pass keys explicitly for anything unusual.

## One to one

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

| Method | Signature | Defaults |
| --- | --- | --- |
| `hasOne` | `hasOne(string $related, ?string $foreignKey = null, ?string $localKey = null)` | FK `{this_table_singular}_id` on the related table; local key = primary key |
| `belongsTo` | `belongsTo(string $related, ?string $foreignKey = null, ?string $ownerKey = null)` | FK `{related_table_singular}_id` on this table; owner key = related primary key |

## One to many

```php
class User extends Model
{
    public function posts(): HasMany
    {
        return $this->hasMany(Post::class, 'author_id');   // explicit FK
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

`hasMany(string $related, ?string $foreignKey = null, ?string $localKey = null)` returns an array (empty when there are no rows).

## Many to many

`belongsToMany` joins through a pivot table. By default the pivot is named from both singular table names in alphabetical order (`roles` + `users` → `role_user`):

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

`$model->pivot` is an array with both pivot keys plus the columns listed in `withPivot()`.

### Attaching, detaching and syncing

```php
$user->roles()->attach(3);                                  // one id
$user->roles()->attach([1, 2]);                             // several
$user->roles()->attach(4, ['granted_at' => date('Y-m-d')]); // extra pivot columns
$user->roles()->attach([5 => ['granted_at' => '2026-11-11']]);

$user->roles()->detach(2);       // returns the number of rows removed
$user->roles()->detach();        // remove all

$changes = $user->roles()->sync([1, 3]);
// ['attached' => [3], 'detached' => [2]]

$user->roles()->sync([4], detaching: false);   // only add
```

`sync()` runs in a transaction. Ids may also be model instances. These methods throw `\LogicException` on an unsaved parent. They do not refresh an already-loaded `$user->roles` — call `$user->refresh()` to reload.

## Has many through

Reach distant rows through an intermediate model — countries → users → posts:

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
    ?string $firstKey = null,        // FK on the through table (country_id)
    ?string $secondKey = null,       // FK on the related table (user_id)
    ?string $localKey = null,        // key on this model (id)
    ?string $secondLocalKey = null,  // key on the through model (id)
)
```

Lazy and eager loading each use one JOIN query.

## Polymorphic relations

One table can belong to several models. A `comments` table with `commentable_type` and `commentable_id` (`$table->morphs('commentable')` in a [migration](migrations.md)) can hold comments for posts and videos:

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
        return $this->morphTo();   // name taken from the method: commentable_type / commentable_id
    }
}

$post->comments;                                   // array<Comment>
$comment->commentable;                             // Post, Video … or null
$post->comments()->create(['body' => 'Nice!']);    // sets type/id, respects $fillable
$post->comments()->save($comment);

$comment->commentable()->associate($video);        // sets type/id …
$comment->save();                                  // … then save
```

| Method | Signature |
| --- | --- |
| `morphMany` / `morphOne` | `(string $related, string $name, ?string $type = null, ?string $id = null, ?string $localKey = null)` |
| `morphTo` | `(?string $name = null, ?string $type = null, ?string $id = null, ?string $ownerKey = null)` |

### Morph map

By default the `_type` column stores the class name. Register short aliases once (for example in a service provider) to decouple stored data from class names:

```php
Model::morphMap([
    'post'  => Post::class,
    'video' => Video::class,
]);
```

Reading an unknown type throws `\UnexpectedValueException`.

## Eager loading

Lazy loading runs one query per model. Eager loading runs one query per relation for all models:

```php
$posts = Post::with('author')->get();                 // 2 queries, not N + 1
$users = User::with('posts', 'profile')->all();
$user  = User::with('roles')->find(1);
```

`Model::with()` returns a `ModelQuery`, so it combines with any query method:

```php
$posts = Post::with('author')->where('published', '=', 1)->paginate(20, $page);
```

### Nested relations

Dot notation loads relations of relations, one query per level:

```php
$users = User::with('posts.comments.author')->get();
```

Nesting works through `morphTo`: children are grouped by class.

### Constraining eager loads

Pass an array of relation => closure. The closure receives a `ModelQuery` of the related model:

```php
use Luany\Database\ModelQuery;

$users = User::with([
    'posts' => fn (ModelQuery $q) => $q->where('published', '=', 1)->orderByDesc('published_at'),
    'roles',
])->get();

// The constraint applies to the last segment
User::with(['posts.comments' => fn (ModelQuery $q) => $q->where('approved', '=', 1)])->get();
```

> **Note** A `limit()` in a constraint limits the whole batched query, not each parent.

## Querying relationship existence

Filter parents by their related rows (a correlated `EXISTS` / `COUNT` subquery, so self-referencing relations work):

```php
// Users with at least one post
User::query()->has('posts')->get();

// Users with at least 3 posts
User::query()->has('posts', '>=', 3)->get();

// Users with a published post with 100+ views
User::query()->whereHas('posts', fn (ModelQuery $q) => $q
    ->where('published', '=', 1)
    ->where('views', '>=', 100))->get();

// Nested
User::query()->whereHas('posts.comments', fn (ModelQuery $q) => $q->where('flagged', '=', 1))->get();

// Without
User::query()->doesntHave('posts')->get();
User::query()->whereDoesntHave('roles', fn (ModelQuery $q) => $q->where('name', '=', 'admin'))->get();

// OR
User::query()->where('vip', '=', 1)->orWhereHas('orders')->get();
```

| Method | Signature |
| --- | --- |
| `whereHas` | `whereHas(string $relation, ?\Closure $callback = null, string $operator = '>=', int $count = 1)` |
| `orWhereHas` | same, OR-ed |
| `whereDoesntHave` | `whereDoesntHave(string $relation, ?\Closure $callback = null)` |
| `has` | `has(string $relation, string $operator = '>=', int $count = 1)` |
| `doesntHave` | `doesntHave(string $relation)` |

Count operators: `=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`. Soft-deleted related rows are excluded. `morphTo` relations do not support existence queries (`\LogicException`).

## Counting related rows

`withCount()` adds a `{relation}_count` attribute without loading the rows:

```php
$users = User::query()
    ->withCount('posts', [
        'posts as published_posts' => fn (ModelQuery $q) => $q->where('published', '=', 1),
    ])
    ->orderByDesc('posts_count')
    ->get();

echo $users[0]->posts_count, ' / ', $users[0]->published_posts;
```

Relation names in camelCase become snake_case (`blogPosts` → `blog_posts_count`). Aliases must be valid identifiers.
