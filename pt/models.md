# Modelos

Os modelos são classes Active Record: cada classe corresponde a uma tabela e cada instância a uma linha. Estendem `Luany\Database\Model` e partilham a ligação registada pelo `DatabaseServiceProvider` do skeleton (ver [Introdução](database.md)).

## Definir modelos

Gere um modelo com a CLI (ou uma funcionalidade CRUD completa com [`make:feature`](make-feature.md)):

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

| Propriedade | Predefinição | Finalidade |
| --- | --- | --- |
| `$table` | `''` | Nome da tabela. Defina-o sempre — não é inferido a partir do nome da classe. |
| `$primaryKey` | `'id'` | Coluna da chave primária. |
| `$incrementing` | `true` | Chave autoincrementada; defina `false` para chaves UUID / string. |
| `$keyType` | `'int'` | Tipo da chave autoincrementada: `int` ou `string`. |
| `$fillable` | `[]` | Colunas permitidas na atribuição em massa. |
| `$guarded` | `['*']` | Colunas bloqueadas quando `$fillable` está vazio. |
| `$hidden` | `[]` | Colunas excluídas de `toArray()` / `toJson()`. |
| `$appends` | `[]` | Atributos que só existem como accessor, acrescentados a `toArray()`. |
| `$casts` | `[]` | Casts dos atributos. |
| `$timestamps` | `false` | Gere `created_at` / `updated_at`. |
| `$dateFormat` | `'Y-m-d H:i:s'` | Formato de armazenamento de datas e timestamps. |

Sem o framework, registe a ligação manualmente: `Model::setConnection($connection)` (uma `Connection` ou uma closure que devolva uma, resolvida na primeira utilização).

## Obter modelos

```php
$post  = Post::find(1);                 // ?Post
$post  = Post::findOrFail(1);           // Post, ou ModelNotFoundException
$posts = Post::all();                   // array<Post>
$posts = Post::all('published_at DESC, id DESC');
$total = Post::count();
```

`all()` só aceita termos `column [ASC|DESC]` separados por vírgulas; tudo o resto lança `\InvalidArgumentException`.

> **Nota** O framework não converte `ModelNotFoundException` num 404. Nos controllers, prefira `Post::find($id) ?? abort(404)`.

### Queries de modelo

`Model::query()` inicia uma `ModelQuery`: todos os métodos do [query builder](query-builder.md), mas devolvendo modelos hidratados.

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

| Método da `ModelQuery` | Devolve |
| --- | --- |
| `get()`, `all(string $orderBy = '')` | `array<Model>` |
| `first()`, `find(int\|string $id)` | `?Model` |
| `firstOrFail()`, `findOrFail($id)` | `Model` ou `ModelNotFoundException` |
| `paginate(int $perPage = 15, int $page = 1)` | [`PaginationResult`](pagination.md) de modelos |
| `pluck(string $column, ?string $key = null)` | Valores com cast, opcionalmente indexados por uma chave |
| `chunk()`, `chunkById()`, `cursor()` | Modelos em lotes / um de cada vez |
| `with()`, `whereHas()`, `withCount()`, … | Ver [Relações](relationships.md) |
| `toBase()` | O `QueryBuilder` subjacente (linhas em bruto) |
| `count()`, `exists()`, `sum()`, `toSql()`, … | Encaminhados para o query builder |

O `update([...])` e o `delete()` em massa numa query de modelo são executados diretamente sobre a tabela: ignoram casts, timestamps e eventos.

### Condições SQL estáticas

`Model::where()`, `firstWhere()` e `count()` recebem uma condição em **SQL bruto** com bindings e são executados de imediato:

```php
$admins = User::where('role = ? AND active = ?', ['admin', 1]);   // array<User>
$user   = User::firstWhere('email = ?', [$email]);                // ?User
$total  = User::count('active = ?', [1]);
```

> **Aviso** `User::where('active', '=', 1)` dá um `TypeError` — o método estático não é o fluente. Use `User::query()->where('active', '=', 1)->get()`. Nunca interpole input na condição em bruto.

## Inserir e atualizar

```php
$post = Post::create(['title' => 'Hello', 'body' => '...']);   // INSERT, devolve o modelo recarregado

$post = new Post();
$post->title = 'Draft';
$post->save();                                                 // INSERT

$post->title = 'Final title';
$post->save();                                                 // UPDATE apenas das colunas alteradas

$post->update(['title' => 'New title']);                       // fill() + save()
```

`save()` devolve `false` quando um listener de eventos o cancelou. Um modelo sem alterações não emite nenhum `UPDATE`. `create()` lança `\InvalidArgumentException` quando nenhuma das chaves indicadas é fillable.

### Atribuição em massa

`fill()`, `create()` e `update()` só aceitam colunas fillable; as restantes chaves são descartadas em silêncio. A atribuição em massa está **fechada por predefinição**:

```php
protected array $fillable = ['title', 'body'];   // lista de permitidas (recomendado)
protected array $guarded  = ['is_admin'];         // ou: tudo exceto estas (com $fillable vazio)
protected array $guarded  = [];                   // ou: tudo — apenas para input de confiança
```

A atribuição direta de propriedades e `forceFill()` contornam esta proteção:

```php
$user->forceFill(['is_admin' => true])->save();
$user->isFillable('is_admin');   // false
```

### Encontrar ou criar

```php
$tag  = Tag::firstOrNew(['slug' => 'php'], ['name' => 'PHP']);     // não guardado quando é novo
$tag  = Tag::firstOrCreate(['slug' => 'php'], ['name' => 'PHP']);   // guardado
$stat = Stat::updateOrCreate(['day' => $today], ['hits' => $hits]);
```

Ambos os arrays passam pela atribuição em massa, por isso as suas chaves têm de ser fillable.

### Recarregar

```php
$copy = $post->fresh();   // nova instância lida da base de dados (?Post); $post fica intacto
$post->refresh();         // recarrega $post no próprio objeto, descartando alterações por guardar e relações em cache
```

## Eliminar

```php
$post->delete();   // false se não estiver persistido ou se um listener "deleting" o cancelar
```

### Soft deletes

Acrescente o trait `SoftDeletes` e uma coluna `deleted_at` que aceite nulos (`$table->softDeletes()` numa [migration](migrations.md)):

```php
use Luany\Database\Concerns\SoftDeletes;

class Post extends Model
{
    use SoftDeletes;
}
```

| Método | Comportamento |
| --- | --- |
| `$post->delete()` | Preenche `deleted_at`; a linha mantém-se. |
| `$post->trashed()` | `true` quando foi eliminado com soft delete. |
| `$post->restore()` | Limpa `deleted_at` (dispara `restoring` / `restored`). |
| `$post->forceDelete()` | Remove a linha definitivamente. |
| `Post::withTrashed(string $orderBy = '')` | `array` com todas as linhas, incluindo as eliminadas. |
| `Post::onlyTrashed(string $orderBy = '')` | `array` apenas com as linhas eliminadas. |

As linhas eliminadas são excluídas de `find()`, `all()`, `query()`, dos métodos estáticos em SQL bruto, das relações e de `whereHas()`. `withTrashed()` e `onlyTrashed()` devolvem arrays, não queries. Redefina `protected string $deletedAtColumn` para usar outra coluna.

## Casts de atributos

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

| Cast | Lido como | Guardado como |
| --- | --- | --- |
| `int`, `integer` | `int` | tal como é dado |
| `float`, `double`, `real` | `float` | tal como é dado |
| `decimal:N` | string com N casas decimais | tal como é dado |
| `bool`, `boolean` | `bool` (`'1'`, `'true'`, `'on'`, `'yes'` são verdadeiros) | `1` / `0` |
| `string` | `string` | tal como é dado |
| `array`, `json` | array descodificado | JSON |
| `datetime`, `immutable_datetime` | `DateTimeImmutable` | `$dateFormat` |
| `date`, `immutable_date` | `DateTimeImmutable` à meia-noite | `Y-m-d` |
| `timestamp` | `int` Unix | `$dateFormat` |

Os casts aplicam-se no acesso a propriedades, em `getAttribute()` e `toArray()` (as datas são serializadas como strings), e na atribuição. Os casts de data aceitam `DateTimeInterface`, timestamps Unix e strings como `2026-11-02T10:30` vindas de `<input type="datetime-local">`; um valor que não possa ser interpretado lança `\InvalidArgumentException`. `getRawAttribute($key)` devolve o valor guardado.

### Casts próprios

Implemente `Luany\Database\Casts\CastsAttributes`:

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

## Accessors, mutators e appends

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

`get{StudlyName}Attribute($value)` recebe o valor já com cast e é executado na leitura; `set{StudlyName}Attribute($value)` é executado na atribuição e tem de escrever ele próprio em `$this->attributes`. `$appends` acrescenta atributos que só existem como accessor a `toArray()` / `toJson()`.

## Serialização

```php
$user->toArray();   // casts aplicados, $hidden removidos, $appends acrescentados
$user->toJson();    // JSON_UNESCAPED_UNICODE por predefinição
```

## Deteção de alterações

```php
$post->title = 'Changed';
$post->isDirty();          // true
$post->isDirty('body');    // false
$post->getDirty();         // ['title' => 'Changed']
$post->getOriginal('title');
$post->exists();           // persistido?
$post->getKey();           // valor da chave primária
```

Valores numericamente iguais (`"5"` e `5`) não são considerados alterações.

## Timestamps

Com `protected bool $timestamps = true;`, o modelo preenche `created_at` e `updated_at` na inserção e `updated_at` em cada atualização com alterações. Renomeie ou desative uma coluna com constantes:

```php
public const CREATED_AT = 'inserted_on';
public const UPDATED_AT = null;
```

`$post->touch()` atualiza apenas a coluna de "atualizado em"; `freshTimestamp()` devolve a hora atual no formato `$dateFormat`.

## Eventos e observers

| Evento | Disparado por |
| --- | --- |
| `saving` / `saved` | cada `save()` |
| `creating` / `created` | inserção |
| `updating` / `updated` | atualização com alterações |
| `deleting` / `deleted` | `delete()`, `forceDelete()` |
| `restoring` / `restored` | `restore()` |

Os listeners recebem o modelo. Devolver `false` num listener `-ing` cancela a operação (o método devolve `false`). Registe-os em `booted()`, que é executado uma vez por classe:

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

Também pode registá-los a partir de qualquer lado — `Post::saved(fn ($post) => …)`, `Post::on('updated', $callable)` — ou associar um observer cujos métodos públicos têm o nome dos eventos:

```php
class PostObserver
{
    public function created(Post $post): void { logger("Post {$post->id} created"); }
}

Post::observe(PostObserver::class);   // um nome de classe ou uma instância
```

`Post::flushEventListeners()` remove os listeners de uma classe. O `update()` / `delete()` em massa de uma query não disparam eventos.

## Scopes locais

Um método `scope{Name}(ModelQuery $query, ...$args)` torna-se um método de query, que pode ser chamado estaticamente ou encadeado:

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

## Chaves UUID e string

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

Use `$table->uuid('id')->primary()` na migration. A chave tem de estar definida antes da inserção: `create()` recarrega a linha pela sua chave.
