# Getting Started

`luany/database` provides a PDO connection wrapper, a fluent query builder, an Active Record model layer with relationships, a portable schema builder, migrations and seeders. It supports **SQLite**, **MySQL** and **PostgreSQL**.

## Configuration

A new project uses SQLite out of the box — no server required. The connection is configured in `.env`:

```env
# sqlite (no server needed) | mysql | pgsql
DB_CONNECTION=sqlite
DB_NAME=database/database.sqlite
```

For MySQL or PostgreSQL:

```env
DB_CONNECTION=mysql
DB_HOST=127.0.0.1
DB_PORT=3306
DB_NAME=luany
DB_USER=root
DB_PASS=secret
```

| Variable | Default | Notes |
| --- | --- | --- |
| `DB_CONNECTION` | `sqlite` | `sqlite`, `mysql` or `pgsql`. |
| `DB_NAME` | `database/database.sqlite` (SQLite), `luany` | SQLite paths are relative to the project root; `:memory:` is accepted. |
| `DB_HOST` | `127.0.0.1` | |
| `DB_PORT` | `3306` / `5432` | Chosen from the driver. |
| `DB_USER` | `root` (MySQL) | |
| `DB_PASS` | empty | |

> **Note** Since luany/cli 1.6, the `luany` database commands (`migrate*`, `db:seed`, `doctor`) use the same default, `sqlite`. Projects on MySQL or PostgreSQL must set `DB_CONNECTION` in `.env`.

The skeleton's `App\Providers\DatabaseServiceProvider` reads these values and registers:

| Container binding | Value |
| --- | --- |
| `connection` | A `Luany\Database\Connection` singleton, opened lazily on the first query. |
| `db` | The raw `\PDO` (used by migrations and the `unique` / `exists` [validation](validation.md) rules). |

In `boot()` it calls `Model::setConnection(fn () => $app->make('connection'))`, so every [model](models.md) shares the connection.

## Creating a connection

Outside the skeleton, create a connection with `Connection::make()`:

```php
use Luany\Database\Connection;

$connection = Connection::make([
    'driver'   => 'mysql',       // mysql (default) | pgsql | sqlite
    'host'     => '127.0.0.1',
    'port'     => '3306',
    'database' => 'shop',
    'username' => 'root',
    'password' => 'secret',
    'charset'  => 'utf8mb4',     // MySQL only
]);

$sqlite = Connection::make(['driver' => 'sqlite', 'database' => __DIR__ . '/app.sqlite']);
$memory = Connection::make(['driver' => 'sqlite']);   // empty database → :memory:
```

The PDO is created with exceptions enabled, `FETCH_ASSOC`, native prepared statements and no stringified fetches. A failed connection throws `\RuntimeException("Database connection failed: …")`; an unknown driver throws `\InvalidArgumentException`.

SQLite connections also run `PRAGMA foreign_keys = ON`, so foreign keys and `ON DELETE` / `ON UPDATE` actions are enforced as on MySQL and PostgreSQL (luany/database 1.8 or newer). This applies to the skeleton's connection too. Pass `'foreign_keys' => false` to opt out:

```php
$legacy = Connection::make(['driver' => 'sqlite', 'database' => 'legacy.sqlite', 'foreign_keys' => false]);
```

`Connection::fromPdo(\PDO $pdo)` wraps an existing PDO — handy in tests. It leaves the PDO's settings untouched, including SQLite's `foreign_keys` pragma.

| Method | Description |
| --- | --- |
| `getPdo(): \PDO` | The underlying PDO. |
| `getDriverName(): string` | `mysql`, `pgsql` or `sqlite`. |
| `getGrammar(): Grammar` | Identifier-quoting grammar for the driver. |
| `execute(string $sql, array $bindings = []): \PDOStatement` | Prepare and execute. On MySQL and SQLite, integers are bound as integers and booleans as `0`/`1`. |
| `lastInsertId(): string` | Last inserted id. |
| `inTransaction(): bool` | Whether a PDO transaction is open. |

## Running queries

Raw SQL goes through `execute()` with placeholders:

```php
$stmt = $connection->execute('SELECT * FROM users WHERE email = ?', [$email]);
$user = $stmt->fetch();
```

Most code uses the [query builder](query-builder.md) or [models](models.md):

```php
use Luany\Database\QueryBuilder;

$db = new QueryBuilder(app('connection'));

$active = $db->table('users')->where('active', '=', 1)->orderBy('name')->get();
```

## Transactions

`transaction()` commits when the callback returns and rolls back when it throws; the callback receives the connection and its return value is passed through:

```php
$orderId = $connection->transaction(function (Connection $db) use ($cart) {
    $db->execute('INSERT INTO orders (total) VALUES (?)', [$cart->total()]);
    $id = (int) $db->lastInsertId();

    foreach ($cart->items() as $item) {
        $db->execute('UPDATE products SET stock = stock - ? WHERE id = ?', [$item->qty, $item->id]);
    }

    return $id;
});
```

The original exception is always rethrown — a failure during rollback never masks it.

### Nested transactions

Nested calls use savepoints (`luany_sp_2`, `luany_sp_3`, …). An inner failure that the outer callback catches undoes only the inner work:

```php
$connection->transaction(function (Connection $db) {
    $db->execute('INSERT INTO audit (event) VALUES (?)', ['import started']);

    try {
        $db->transaction(fn () => importRows($db));   // rolled back to its savepoint on failure
    } catch (\Throwable $e) {
        logger('Import failed: ' . $e->getMessage());
    }
});   // the audit row is committed either way
```

Manual control is available too:

| Method | At level 0 | When nested |
| --- | --- | --- |
| `beginTransaction()` | `BEGIN` | `SAVEPOINT` |
| `commit()` | `COMMIT` | `RELEASE SAVEPOINT` |
| `rollBack()` | `ROLLBACK` | `ROLLBACK TO SAVEPOINT` |
| `transactionLevel()` | `0` | nesting depth |

> **Note** MySQL commits DDL (`CREATE TABLE`, `ALTER TABLE`) implicitly. Do not rely on rolling back schema changes on MySQL.

## Next steps

- [Query Builder](query-builder.md) — fluent, injection-safe SQL.
- [Models](models.md) and [Relationships](relationships.md).
- [Migrations & Schema](migrations.md) and [Seeding](seeding.md).
