# Models

Models are Active Record classes: each class maps to a table and each instance to a row. They extend `Luany\Database\Model` and share the connection registered by the skeleton's `DatabaseServiceProvider` (see [Getting Started](database.md)).

## Defining models

Generate a model with the CLI (or a full CRUD feature with [`make:feature`](make-feature.md)):

```bash
luany make:model Post
```

```php
namespace App\Models;

use Luany\Database\Model;

class Post extends Model
{
    protected string $table = 'posts';

    protected bool $timestamps = true;

    protected array $fillable = ['title', 'body', 'published_at'];

    protected array $hidden = [];
}
```

| Property | Default | Purpose |
| --- | --- | --- |
| `$table` | `''` | Table name. Always set it — it is not inferred from the class name. |
| `$primaryKey` | `'id'` | Primary key column. |
| `$incrementing` | `true` | Auto-incrementing key; set `false` for UUID / string keys. |
| `$keyType` | `'int'` | Type of the auto-incremented key: `int` or `string`. |
| `$fillable` | `[]` | Columns allowed for mass assignment. |
| `$guarded` | `['*']` | Columns blocked when `$fillable` is empty. |
| `$hidden` | `[]` | Columns left out of `toArray()` / `toJson()`. |
| `$appends` | `[]` | Accessor-only attributes added to `toArray()`. |
| `$casts` | `[]` | Attribute casts. |
| `$timestamps` | `false` | Manage `created_at` / `updated_at`. |
| `$dateFormat` | `'Y-m-d H:i:s'` | Storage format for dates and timestamps. |

Without a framework, register the connection yourself: `Model::setConnection($connection)` (a `Connection` or a closure returning one, resolved on first use).

## Retrieving models

```php
$post  = Post::find(1);                 // ?Post
$post  = Post::findOrFail(1);           // Post, or ModelNotFoundException
$posts = Post::all();                   // array<Post>
$posts = Post::all('published_at DESC, id DESC');
$total = Post::count();
```

`all()` accepts only `column [ASC|DESC]` terms separated by commas; anything else throws `\InvalidArgumentException`.

> **Note** `ModelNotFoundException` is not converted to a 404 by the framework. In controllers, prefer `Post::find($id) ?? abort(404)`.

### Model queries

`Model::query()` starts a `ModelQuery`: every [query builder](query-builder.md) method, returning hydrated models.

```php
$posts = Post::query()
    ->where('published', '=', 1)
    ->where('views', '>', 100)
    ->orderByDesc('published_at')
    ->limit(10)
    ->get();                              // array<Post>

$latest = Post::query()->orderByDesc('id')->first();          // ?Post
$post   = Post::query()->where('slug', '=', $slug)->firstOrFail();
$titles = Post::query()->pluck('title');                      // ['Hello', …]
$byId   = Post::query()->pluck('title', 'id');                // [1 => 'Hello', …]
$views  = Post::query()->where('published', '=', 1)->sum('views');
```

| `ModelQuery` method | Returns |
| --- | --- |
| `get()`, `all(string $orderBy = '')` | `array<Model>` |
| `first()`, `find(int\|string $id)` | `?Model` |
| `firstOrFail()`, `findOrFail($id)` | `Model` or `ModelNotFoundException` |
| `paginate(int $perPage = 15, int $page = 1)` | [`PaginationResult`](pagination.md) of models |
| `pluck(string $column, ?string $key = null)` | Cast values, optionally keyed |
| `chunk()`, `chunkById()`, `cursor()` | Models in batches / one at a time |
| `with()`, `whereHas()`, `withCount()`, … | See [Relationships](relationships.md) |
| `toBase()` | The underlying `QueryBuilder` (raw rows) |
| `count()`, `exists()`, `sum()`, `toSql()`, … | Forwarded to the query builder |

Bulk `update([...])` and `delete()` on a model query run directly on the table: they skip casts, timestamps and events.

### Static raw conditions

`Model::where()`, `firstWhere()` and `count()` take a **raw SQL** condition with bindings and run immediately:

```php
$admins = User::where('role = ? AND active = ?', ['admin', 1]);   // array<User>
$user   = User::firstWhere('email = ?', [$email]);                // ?User
$total  = User::count('active = ?', [1]);
```

> **Warning** `User::where('active', '=', 1)` is a `TypeError` — the static method is not the fluent one. Use `User::query()->where('active', '=', 1)->get()`. Never interpolate input into the raw condition.

## Inserting and updating

```php
$post = Post::create(['title' => 'Hello', 'body' => '...']);   // INSERT, returns the model reloaded

$post = new Post();
$post->title = 'Draft';
$post->save();                                                 // INSERT

$post->title = 'Final title';
$post->save();                                                 // UPDATE of changed columns only

$post->update(['title' => 'New title']);                       // fill() + save()
```

`save()` returns `false` when an event listener cancelled it. An unchanged model issues no `UPDATE`. `create()` throws `\InvalidArgumentException` when none of the given keys is fillable.

### Mass assignment

`fill()`, `create()` and `update()` accept only fillable columns; other keys are silently dropped. Mass assignment is **closed by default**:

```php
protected array $fillable = ['title', 'body'];   // allow-list (recommended)
protected array $guarded  = ['is_admin'];         // or: everything except these (with empty $fillable)
protected array $guarded  = [];                   // or: everything — trusted input only
```

Direct property assignment and `forceFill()` bypass the guard:

```php
$user->forceFill(['is_admin' => true])->save();
$user->isFillable('is_admin');   // false
```

### Finding or creating

```php
$tag  = Tag::firstOrNew(['slug' => 'php'], ['name' => 'PHP']);     // unsaved when new
$tag  = Tag::firstOrCreate(['slug' => 'php'], ['name' => 'PHP']);   // saved
$stat = Stat::updateOrCreate(['day' => $today], ['hits' => $hits]);
```

Both arrays go through mass assignment, so their keys must be fillable.

### Reloading

```php
$copy = $post->fresh();   // new instance from the database (?Post); $post untouched
$post->refresh();         // reload $post in place, dropping unsaved changes and cached relations
```

## Deleting

```php
$post->delete();   // false if not persisted or cancelled by a "deleting" listener
```

### Soft deletes

Add the `SoftDeletes` trait and a nullable `deleted_at` column (`$table->softDeletes()` in a [migration](migrations.md)):

```php
use Luany\Database\Concerns\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;
}
```

| Method | Behaviour |
| --- | --- |
| `$post->delete()` | Sets `deleted_at`; the row stays. |
| `$post->trashed()` | `true` when soft-deleted. |
| `$post->restore()` | Clears `deleted_at` (fires `restoring` / `restored`). |
| `$post->forceDelete()` | Removes the row permanently. |
| `Post::withTrashed(string $orderBy = '')` | `array` of all rows, including trashed. |
| `Post::onlyTrashed(string $orderBy = '')` | `array` of trashed rows only. |

Trashed rows are excluded from `find()`, `all()`, `query()`, the static raw methods, relationships and `whereHas()`. `withTrashed()` and `onlyTrashed()` return arrays, not queries. Override `protected string $deletedAtColumn` to use another column.

## Attribute casting

```php
protected array $casts = [
    'active'       => 'bool',
    'qty'          => 'int',
    'price'        => 'decimal:2',
    'options'      => 'array',
    'published_at' => 'datetime',
    'birthday'     => 'date',
];
```

| Cast | Read as | Stored as |
| --- | --- | --- |
| `int`, `integer` | `int` | as given |
| `float`, `double`, `real` | `float` | as given |
| `decimal:N` | string with N decimals | as given |
| `bool`, `boolean` | `bool` (`'1'`, `'true'`, `'on'`, `'yes'` are true) | `1` / `0` |
| `string` | `string` | as given |
| `array`, `json` | decoded array | JSON |
| `datetime`, `immutable_datetime` | `DateTimeImmutable` | `$dateFormat` |
| `date`, `immutable_date` | `DateTimeImmutable` at midnight | `Y-m-d` |
| `timestamp` | Unix `int` | `$dateFormat` |

Casts apply on property access, `getAttribute()` and `toArray()` (dates are serialised as strings), and on assignment. Date casts accept `DateTimeInterface`, Unix timestamps and strings such as `2026-11-02T10:30` from `<input type="datetime-local">`; unparseable input throws `\InvalidArgumentException`. `getRawAttribute($key)` returns the stored value.

### Custom casts

Implement `Luany\Database\Casts\CastsAttributes`:

```php
use Luany\Database\Casts\CastsAttributes;
use Luany\Database\Model;

final class MoneyCast implements CastsAttributes
{
    public function get(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return $value === null ? null : new Money((int) $value);
    }

    public function set(Model $model, string $key, mixed $value, array $attributes): mixed
    {
        return $value instanceof Money ? $value->cents : $value;
    }
}
```

```php
protected array $casts = ['price' => MoneyCast::class];
```

## Accessors, mutators and appends

```php
class User extends Model
{
    protected array $appends = ['full_name'];

    public function getFullNameAttribute(): string
    {
        return trim($this->first_name . ' ' . $this->last_name);
    }

    public function setEmailAttribute(string $value): void
    {
        $this->attributes['email'] = strtolower(trim($value));
    }
}
```

`get{StudlyName}Attribute($value)` receives the cast value and runs on read; `set{StudlyName}Attribute($value)` runs on assignment and must write `$this->attributes` itself. `$appends` adds accessor-only attributes to `toArray()` / `toJson()`.

## Serialisation

```php
$user->toArray();   // casts applied, $hidden removed, $appends added
$user->toJson();    // JSON_UNESCAPED_UNICODE by default
```

## Dirty tracking

```php
$post->title = 'Changed';
$post->isDirty();          // true
$post->isDirty('body');    // false
$post->getDirty();         // ['title' => 'Changed']
$post->getOriginal('title');
$post->exists();           // persisted?
$post->getKey();           // primary key value
```

Numerically equal values (`"5"` and `5`) are not considered changes.

## Timestamps

With `protected bool $timestamps = true;` the model sets `created_at` and `updated_at` on insert and `updated_at` on every update with changes. Rename or disable a column with constants:

```php
public const CREATED_AT = 'inserted_on';
public const UPDATED_AT = null;
```

`$post->touch()` updates only the "updated at" column; `freshTimestamp()` returns the current time in `$dateFormat`.

## Events and observers

| Event | Fired by |
| --- | --- |
| `saving` / `saved` | every `save()` |
| `creating` / `created` | insert |
| `updating` / `updated` | update with changes |
| `deleting` / `deleted` | `delete()`, `forceDelete()` |
| `restoring` / `restored` | `restore()` |

Listeners receive the model. Returning `false` from an `-ing` listener cancels the operation (the method returns `false`). Register them in `booted()`, which runs once per class:

```php
class Post extends Model
{
    protected static function booted(): void
    {
        static::creating(fn (Post $post) => $post->slug ??= strtolower(str_replace(' ', '-', $post->title)));
        static::deleting(fn (Post $post) => !$post->locked);
    }
}
```

Or register from anywhere — `Post::saved(fn ($post) => …)`, `Post::on('updated', $callable)` — or attach an observer whose public methods are named after events:

```php
class PostObserver
{
    public function created(Post $post): void { logger("Post {$post->id} created"); }
}

Post::observe(PostObserver::class);   // a class name or an instance
```

`Post::flushEventListeners()` removes a class's listeners. Bulk query `update()` / `delete()` do not fire events.

## Local scopes

A `scope{Name}(ModelQuery $query, ...$args)` method becomes a query method, callable statically or chained:

```php
use Luany\Database\ModelQuery;

class Post extends Model
{
    public function scopePublished(ModelQuery $query): void
    {
        $query->where('published', '=', 1);
    }

    public function scopeOfType(ModelQuery $query, string $type): void
    {
        $query->where('type', '=', $type);
    }
}

$posts = Post::published()->ofType('news')->orderByDesc('id')->get();
```

## UUID and string keys

```php
class Order extends Model
{
    protected string $table = 'orders';
    protected bool $incrementing = false;
    protected array $fillable = ['total'];

    protected static function booted(): void
    {
        static::creating(function (Order $order) {
            $order->id ??= self::uuid();
        });
    }

    private static function uuid(): string
    {
        $b = random_bytes(16);
        $b[6] = chr(ord($b[6]) & 0x0f | 0x40);
        $b[8] = chr(ord($b[8]) & 0x3f | 0x80);
        return vsprintf('%s%s-%s-%s-%s-%s%s%s', str_split(bin2hex($b), 4));
    }
}
```

Use `$table->uuid('id')->primary()` in the migration. The key must be set before the insert: `create()` reloads the row by its key.
