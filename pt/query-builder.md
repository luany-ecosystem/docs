# Query builder

`Luany\Database\QueryBuilder` constrói SQL de forma fluente. Os valores são sempre associados como parâmetros; os nomes de tabelas e colunas são validados e citados de acordo com o driver ativo. Os resultados são arrays associativos simples — use [modelos](models.md) quando quiser objetos.

## Obter um builder

```php
use Luany\Database\QueryBuilder;

$db = new QueryBuilder(app('connection'));   // ou qualquer Connection

$users = $db->table('users')->get();
```

`table()` devolve um builder **novo** de cada vez, por isso um único `$db` pode iniciar quantas queries independentes quiser. Chamar um método terminal antes de `table()` lança `\RuntimeException`.

## Obter linhas

```php
$rows  = $db->table('users')->get();                       // array<int, array<string, mixed>>
$row   = $db->table('users')->where('id', '=', 7)->first(); // ?array
$count = $db->table('users')->count();                      // int
$any   = $db->table('users')->where('active', '=', 1)->exists(); // bool
```

> **Nota** O query builder não tem `find()` nem `pluck()`. Use `where('id', '=', $id)->first()`, ou uma query de modelo (`User::query()->pluck('name')`).

### Selecionar colunas

```php
$db->table('users')->select('id', 'name', 'email AS contact')->get();
$db->table('users')->distinct()->select('country')->get();

$db->table('orders')
    ->selectRaw('status, COUNT(*) AS total, SUM(amount) AS revenue')
    ->groupBy('status')
    ->get();
```

`selectRaw(string $expression, array $bindings = [])` acrescenta uma expressão sem escape; os seus bindings vêm antes dos bindings do `WHERE`.

## Cláusulas where

`where()` recebe sempre **três** argumentos — coluna, operador, valor:

```php
$db->table('posts')
    ->where('published', '=', 1)
    ->where('views', '>=', 100)
    ->orWhere('featured', '=', 1)
    ->get();
```

Comparar com `null` usando `=`, `!=` ou `<>` compila para `IS NULL` / `IS NOT NULL`.

| Método | SQL |
| --- | --- |
| `where($col, $op, $value)` / `orWhere(…)` | `col op ?` |
| `whereIn($col, array $values)` | `col IN (?, ?)` — um array vazio não corresponde a nada |
| `whereNotIn($col, array $values)` | `col NOT IN (…)` — um array vazio corresponde a tudo |
| `whereNull($col)` / `whereNotNull($col)` | `col IS [NOT] NULL` |
| `whereColumn($first, $op, $second, $boolean = 'AND')` | `first op second` (coluna com coluna) |
| `whereNested(\Closure $cb, $boolean = 'AND')` / `orWhereNested($cb)` | grupo `( … )` |
| `whereRaw($sql, array $bindings = [], $boolean = 'AND')` / `orWhereRaw($sql, $bindings)` | `(sql)` — sem escape |

### Agrupar condições

```php
// WHERE active = ? AND (role = ? OR role = ?)
$db->table('users')
    ->where('active', '=', 1)
    ->whereNested(fn (QueryBuilder $q) => $q
        ->where('role', '=', 'admin')
        ->orWhere('role', '=', 'editor'))
    ->get();
```

### Condições em SQL bruto

```php
$db->table('events')
    ->whereRaw('starts_at BETWEEN ? AND ?', [$from, $to])
    ->get();
```

> **Aviso** `whereRaw()`, `orWhereRaw()`, `selectRaw()`, `havingRaw()` e `orderByRaw()` inserem o SQL tal como está. Use placeholders `?` para todos os valores e nunca concatene input do utilizador.

## Segurança dos identificadores e operadores

Todas as tabelas, colunas, aliases e chaves de dados passam pela gramática do driver. Formas aceites:

| Forma | Citada como (MySQL / SQLite) | PostgreSQL |
| --- | --- | --- |
| `column` | `` `column` `` | `"column"` |
| `table.column` | `` `table`.`column` `` | `"table"."column"` |
| `*`, `table.*` | `*`, `` `table`.* `` | |
| `column AS alias` | `` `column` AS `alias` `` | |

Cada segmento só pode conter letras, dígitos e underscores e não pode começar por um dígito. Tudo o resto — `COUNT(*)`, `id; DROP TABLE` — lança `\InvalidArgumentException`; passe as expressões para os métodos `*Raw()`.

Os operadores estão numa lista de permitidos (sem distinção entre maiúsculas e minúsculas): `=`, `!=`, `<>`, `<`, `<=`, `>`, `>=`, `LIKE`, `NOT LIKE`. Os restantes lançam `\InvalidArgumentException`.

## Ordenação, limit e offset

```php
$db->table('posts')
    ->orderBy('published_at', 'DESC')   // direção: ASC (predefinida) ou DESC
    ->orderBy('id')                     // várias chamadas acumulam-se
    ->limit(10)
    ->offset(20)
    ->get();

$db->table('posts')->orderByDesc('views')->first();
$db->table('posts')->orderByRaw('RANDOM()')->limit(3)->get();
```

## Joins e aliases

Os joins comparam duas colunas. As tabelas aceitam um alias com `AS`:

```php
$rows = $db->table('posts AS p')
    ->join('users AS u', 'u.id', '=', 'p.user_id')
    ->leftJoin('categories AS c', 'c.id', '=', 'p.category_id')
    ->select('p.title', 'u.name AS author', 'c.name AS category')
    ->where('p.published', '=', 1)
    ->get();
```

`join(string $table, string $first, string $operator, string $second, string $type = 'INNER')` aceita `INNER`, `LEFT`, `RIGHT` ou `CROSS`; `leftJoin()` é um atalho. `alias(string $alias)` atribui um alias à tabela atual depois de `table()`.

## Agrupamento e having

```php
$db->table('orders')
    ->selectRaw('customer_id, SUM(amount) AS spent')
    ->groupBy('customer_id')
    ->havingRaw('SUM(amount) > ?', [1000])
    ->get();
```

`having(string $column, string $operator, mixed $value, string $boolean = 'AND')` compara uma coluna ou um alias de `selectRaw()`. Os aliases no `HAVING` funcionam em MySQL e SQLite; em PostgreSQL use `havingRaw()`.

## Agregados

```php
$db->table('orders')->where('status', '=', 'paid')->sum('amount'); // int|float|null
$db->table('orders')->avg('amount');                               // ?float
$db->table('products')->min('price');                              // mixed
$db->table('products')->max('price');                              // mixed
$db->table('users')->count();                                      // int
```

`sum()`, `avg()`, `min()` e `max()` devolvem `null` quando nenhuma linha corresponde e lançam `\LogicException` quando combinados com `groupBy()` — nesse caso use `selectRaw('SUM(x) AS total')->groupBy(…)->get()`. `count()` (e `paginate()`) respeitam joins, `DISTINCT` e `GROUP BY`.

## Inserir

```php
$db->table('users')->insert(['name' => 'Ana', 'email' => 'ana@example.com']);
$id = $db->lastInsertId();

// Em lote: uma só instrução; todas as linhas têm de ter as mesmas chaves
$db->table('tags')->insert([
    ['name' => 'php',  'slug' => 'php'],
    ['name' => 'lte',  'slug' => 'lte'],
]);
```

`insert()` devolve `true`. Um array vazio insere uma linha com os valores predefinidos.

### Upserts

`upsert(array $rows, array $uniqueBy, ?array $update = null): int` insere linhas e atualiza-as quando há conflito num índice `PRIMARY KEY` / `UNIQUE` sobre `$uniqueBy`:

```php
$db->table('products')->upsert(
    [
        ['sku' => 'A1', 'name' => 'Mug',   'price' => 9.5],
        ['sku' => 'B2', 'name' => 'Shirt', 'price' => 19],
    ],
    uniqueBy: ['sku'],
    update:   ['price'],        // predefinição: todas as colunas exceto uniqueBy
);
```

| Driver | SQL |
| --- | --- |
| MySQL | `ON DUPLICATE KEY UPDATE col = VALUES(col)` |
| PostgreSQL, SQLite | `ON CONFLICT (sku) DO UPDATE SET col = excluded.col` |

Com `update: []`, as linhas existentes ficam intactas (`DO NOTHING` / uma atualização sem efeito em MySQL). O valor devolvido é a contagem de linhas afetadas reportada pelo driver. Também é aceite uma única linha associativa.

## Atualizar e eliminar

```php
$affected = $db->table('users')->where('id', '=', 7)->update(['name' => 'Ana Silva']);
$deleted  = $db->table('sessions')->where('expires_at', '<', date('Y-m-d H:i:s'))->delete();
```

Ambos devolvem o número de linhas afetadas. Lançam `\LogicException` numa query com joins — filtre antes com `whereIn()`.

> **Aviso** `update()` e `delete()` sem `where` afetam todas as linhas da tabela.

## Grandes conjuntos de resultados

```php
// Páginas de 500 linhas com LIMIT/OFFSET, pela ordem da query
$db->table('users')->orderBy('id')->chunk(500, function (array $rows, int $page) {
    foreach ($rows as $row) { /* ... */ }
    // devolva false para parar
});

// Paginação por chave (WHERE id > último ORDER BY id): segura quando o callback atualiza ou elimina linhas
$db->table('users')->where('active', '=', 0)->chunkById(500, function (array $rows) use ($db) {
    $db->table('users')->whereIn('id', array_column($rows, 'id'))->delete();
});

// Uma linha de cada vez com um generator
foreach ($db->table('logs')->cursor() as $row) {
    // ...
}
```

`chunk()` e `chunkById(int $size, callable $callback, string $column = 'id')` devolvem `false` quando o callback parou mais cedo e `true` caso contrário. Um tamanho inferior a 1 lança uma exceção.

## Paginação

`paginate(int $perPage = 15, int $page = 1)` devolve um `PaginationResult`. Ver [Paginação](pagination.md).

## Depuração e SQL bruto

```php
$q = $db->table('users')->where('email', 'LIKE', '%@example.com');

$q->toSql();       // SELECT * FROM `users` WHERE `email` LIKE ?
$q->getBindings(); // ['%@example.com']
```

| Método | Devolve |
| --- | --- |
| `query(string $sql, array $bindings = []): Result` / `raw(…)` | Um `Result` com `fetchAll()`, `fetchOne()`, `fetchColumn(int $column = 0)`, `fetchAllAs(string $class)`, `rowCount()`. |
| `statement(string $sql, array $bindings = []): int` | Linhas afetadas (INSERT / UPDATE / DELETE). |
| `lastInsertId(): string` | Último id inserido. |
| `getConnection(): Connection` | A ligação. |

```php
$emails = $db->query('SELECT email FROM users WHERE active = ?', [1])->fetchColumn();
```

> **Nota** `first()` define `LIMIT 1` no builder em que é chamado. Inicie uma nova query com `table()` em vez de reutilizar um builder depois de `first()`.
