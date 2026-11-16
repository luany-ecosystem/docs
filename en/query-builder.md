# Query Builder

`Luany\Database\QueryBuilder` builds SQL fluently. Values are always bound as parameters; table and column names are validated and quoted for the active driver. Results are plain associative arrays — use [models](models.md) when you want objects.

## Getting a builder

```php
use Luany\Database\QueryBuilder;

$db = new QueryBuilder(app('connection'));   // or any Connection

$users = $db->table('users')->get();
```

`table()` returns a **fresh** builder each time, so one `$db` can start any number of independent queries. Calling a terminating method before `table()` throws `\RuntimeException`.

## Retrieving rows

```php
$rows  = $db->table('users')->get();                       // array<int, array<string, mixed>>
$row   = $db->table('users')->where('id', '=', 7)->first(); // ?array
$count = $db->table('users')->count();                      // int
$any   = $db->table('users')->where('active', '=', 1)->exists(); // bool
```

> **Note** There is no `find()` or `pluck()` on the query builder. Use `where('id', '=', $id)->first()`, or a model query (`User::query()->pluck('name')`).

### Selecting columns

```php
$db->table('users')->select('id', 'name', 'email AS contact')->get();
$db->table('users')->distinct()->select('country')->get();

$db->table('orders')
    ->selectRaw('status, COUNT(*) AS total, SUM(amount) AS revenue')
    ->groupBy('status')
    ->get();
```

`selectRaw(string $expression, array $bindings = [])` adds an unescaped expression; its bindings come before the `WHERE` bindings.

## Where clauses

`where()` always takes **three** arguments — column, operator, value:

```php
$db->table('posts')
    ->where('published', '=', 1)
    ->where('views', '>=', 100)
    ->orWhere('featured', '=', 1)
    ->get();
```

Comparing with `null` using `=`, `!=` or `<>` compiles to `IS NULL` / `IS NOT NULL`.

| Method | SQL |
| --- | --- |
| `where($col, $op, $value)` / `orWhere(…)` | `col op ?` |
| `whereIn($col, array $values)` | `col IN (?, ?)` — an empty array matches nothing |
| `whereNotIn($col, array $values)` | `col NOT IN (…)` — an empty array matches everything |
| `whereNull($col)` / `whereNotNull($col)` | `col IS [NOT] NULL` |
| `whereColumn($first, $op, $second, $boolean = 'AND')` | `first op second` (column to column) |
| `whereNested(\Closure $cb, $boolean = 'AND')` / `orWhereNested($cb)` | `( … )` group |
| `whereRaw($sql, array $bindings = [], $boolean = 'AND')` / `orWhereRaw($sql, $bindings)` | `(sql)` — not escaped |

### Grouping conditions

```php
// WHERE active = ? AND (role = ? OR role = ?)
$db->table('users')
    ->where('active', '=', 1)
    ->whereNested(fn (QueryBuilder $q) => $q
        ->where('role', '=', 'admin')
        ->orWhere('role', '=', 'editor'))
    ->get();
```

### Raw conditions

```php
$db->table('events')
    ->whereRaw('starts_at BETWEEN ? AND ?', [$from, $to])
    ->get();
```

> **Warning** `whereRaw()`, `orWhereRaw()`, `selectRaw()`, `havingRaw()` and `orderByRaw()` insert SQL verbatim. Use `?` placeholders for every value and never concatenate user input.

## Identifier safety and operators

Every table, column, alias and data key passes through the driver grammar. Accepted forms:

| Form | Quoted as (MySQL / SQLite) | PostgreSQL |
| --- | --- | --- |
| `column` | `` `column` `` | `"column"` |
| `table.column` | `` `table`.`column` `` | `"table"."column"` |
| `*`, `table.*` | `*`, `` `table`.* `` | |
| `column AS alias` | `` `column` AS `alias` `` | |

Segments may contain only letters, digits and underscores and must not start with a digit. Anything else — `COUNT(*)`, `id; DROP TABLE` — throws `\InvalidArgumentException`; move expressions to the `*Raw()` methods.

Operators are whitelisted (case-insensitive): `=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`, `LIKE`, `NOT LIKE`. Others throw `\InvalidArgumentException`.

## Ordering, limit and offset

```php
$db->table('posts')
    ->orderBy('published_at', 'DESC')   // direction: ASC (default) or DESC
    ->orderBy('id')                     // multiple calls append
    ->limit(10)
    ->offset(20)
    ->get();

$db->table('posts')->orderByDesc('views')->first();
$db->table('posts')->orderByRaw('RANDOM()')->limit(3)->get();
```

## Joins and aliases

Joins compare two columns. Tables accept an alias with `AS`:

```php
$rows = $db->table('posts AS p')
    ->join('users AS u', 'u.id', '=', 'p.user_id')
    ->leftJoin('categories AS c', 'c.id', '=', 'p.category_id')
    ->select('p.title', 'u.name AS author', 'c.name AS category')
    ->where('p.published', '=', 1)
    ->get();
```

`join(string $table, string $first, string $operator, string $second, string $type = 'INNER')` accepts `INNER`, `LEFT`, `RIGHT` or `CROSS`; `leftJoin()` is a shortcut. `alias(string $alias)` aliases the current table after `table()`.

## Grouping and having

```php
$db->table('orders')
    ->selectRaw('customer_id, SUM(amount) AS spent')
    ->groupBy('customer_id')
    ->havingRaw('SUM(amount) > ?', [1000])
    ->get();
```

`having(string $column, string $operator, mixed $value, string $boolean = 'AND')` compares a column or `selectRaw()` alias. Aliases in `HAVING` work on MySQL and SQLite; on PostgreSQL use `havingRaw()`.

## Aggregates

```php
$db->table('orders')->where('status', '=', 'paid')->sum('amount'); // int|float|null
$db->table('orders')->avg('amount');                               // ?float
$db->table('products')->min('price');                              // mixed
$db->table('products')->max('price');                              // mixed
$db->table('users')->count();                                      // int
```

`sum()`, `avg()`, `min()` and `max()` return `null` when no rows match and throw `\LogicException` when combined with `groupBy()` — use `selectRaw('SUM(x) AS total')->groupBy(…)->get()` instead. `count()` (and `paginate()`) honour joins, `DISTINCT` and `GROUP BY`.

## Inserting

```php
$db->table('users')->insert(['name' => 'Ana', 'email' => 'ana@example.com']);
$id = $db->lastInsertId();

// Batch: one statement; every row must have the same keys
$db->table('tags')->insert([
    ['name' => 'php',  'slug' => 'php'],
    ['name' => 'lte',  'slug' => 'lte'],
]);
```

`insert()` returns `true`. An empty array inserts a row of defaults.

### Upserts

`upsert(array $rows, array $uniqueBy, ?array $update = null): int` inserts rows and updates them when a `PRIMARY KEY` / `UNIQUE` index on `$uniqueBy` conflicts:

```php
$db->table('products')->upsert(
    [
        ['sku' => 'A1', 'name' => 'Mug',   'price' => 9.5],
        ['sku' => 'B2', 'name' => 'Shirt', 'price' => 19],
    ],
    uniqueBy: ['sku'],
    update:   ['price'],        // default: every column except uniqueBy
);
```

| Driver | SQL |
| --- | --- |
| MySQL | `ON DUPLICATE KEY UPDATE col = VALUES(col)` |
| PostgreSQL, SQLite | `ON CONFLICT (sku) DO UPDATE SET col = excluded.col` |

With `update: []` existing rows are left untouched (`DO NOTHING` / a no-op update on MySQL). The return value is the driver's affected-row count. A single associative row is accepted too.

## Updating and deleting

```php
$affected = $db->table('users')->where('id', '=', 7)->update(['name' => 'Ana Silva']);
$deleted  = $db->table('sessions')->where('expires_at', '<', date('Y-m-d H:i:s'))->delete();
```

Both return the number of affected rows. They throw `\LogicException` on a query with joins — filter with `whereIn()` instead.

> **Warning** `update()` and `delete()` without a `where` affect every row of the table.

## Large result sets

```php
// Pages of 500 rows using LIMIT/OFFSET in the query's order
$db->table('users')->orderBy('id')->chunk(500, function (array $rows, int $page) {
    foreach ($rows as $row) { /* ... */ }
    // return false to stop
});

// Keyset pagination (WHERE id > last ORDER BY id): safe when the callback updates or deletes rows
$db->table('users')->where('active', '=', 0)->chunkById(500, function (array $rows) use ($db) {
    $db->table('users')->whereIn('id', array_column($rows, 'id'))->delete();
});

// One row at a time with a generator
foreach ($db->table('logs')->cursor() as $row) {
    // ...
}
```

`chunk()` and `chunkById(int $size, callable $callback, string $column = 'id')` return `false` when the callback stopped early, `true` otherwise. A size below 1 throws.

## Pagination

`paginate(int $perPage = 15, int $page = 1)` returns a `PaginationResult`. See [Pagination](pagination.md).

## Debugging and raw SQL

```php
$q = $db->table('users')->where('email', 'LIKE', '%@example.com');

$q->toSql();       // SELECT * FROM `users` WHERE `email` LIKE ?
$q->getBindings(); // ['%@example.com']
```

| Method | Returns |
| --- | --- |
| `query(string $sql, array $bindings = []): Result` / `raw(…)` | A `Result` with `fetchAll()`, `fetchOne()`, `fetchColumn(int $column = 0)`, `fetchAllAs(string $class)`, `rowCount()`. |
| `statement(string $sql, array $bindings = []): int` | Affected rows (INSERT / UPDATE / DELETE). |
| `lastInsertId(): string` | Last inserted id. |
| `getConnection(): Connection` | The connection. |

```php
$emails = $db->query('SELECT email FROM users WHERE active = ?', [1])->fetchColumn();
```

> **Note** `first()` leaves the builder unchanged (luany/database 1.8 or newer): a later `get()` on the same builder still returns every row. The same holds for `first()` and `find()` on [model queries](models.md).
